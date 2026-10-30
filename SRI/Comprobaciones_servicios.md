# Comprobaciones rápidas: qué servicios hay, dónde están y si funcionan

> Para cuando alguien pregunte: *¿qué servicios tienes activos, dónde, funcionan, me lo demuestras?* Un resumen, dos scripts (una tabla para enseñar y una comprobación detallada), y los comandos sueltos por si hay que enseñar algo concreto.

## Índice

- [Qué hay montado](#qué-hay-montado)
- [Tabla de servicios con su dominio](#tabla-de-servicios-con-su-dominio)
- [Demostración detallada en un solo comando](#demostración-detallada-en-un-solo-comando)
- [Comprobaciones sueltas](#comprobaciones-sueltas)
  - [Servicios activos](#servicios-activos)
  - [DNS](#dns)
  - [DHCP](#dhcp)
  - [Web](#web)
  - [Firewall](#firewall)
  - [Desde un cliente](#desde-un-cliente)
- [Demostrar la redundancia del DNS](#demostrar-la-redundancia-del-dns)
- [Si algo sale mal](#si-algo-sale-mal)

## Qué hay montado

| Servicio | Programa | Máquina | Nombre |
|---|---|---|---|
| DNS primario | BIND9 | `server` (`172.16.6.20`) | `dns.jordi.asir2.iespoveda.local` |
| DNS secundario | BIND9 | `server2` (`172.16.6.21`) | `dns2.jordi.asir2.iespoveda.local` (recibe las zonas del primario) |
| DHCP primario | Kea (HA, hot-standby) | `server` (`172.16.6.20`) | `dhcp.jordi.asir2.iespoveda.local` |
| DHCP secundario | Kea (HA, en espera) | `server2` (`172.16.6.21`) | `dhcp2.jordi.asir2.iespoveda.local` |
| Web principal | Nginx | `server` (`172.16.6.20`) | `www.jordi.asir2.iespoveda.local` |
| Web secundaria | Nginx | `server2` (`172.16.6.21`) | `www2.jordi.asir2.iespoveda.local` |

Red `172.16.6.0/24`, puerta de enlace `172.16.6.1`, pool DHCP `172.16.6.200 - 172.16.6.250`. Los clientes reciben por DHCP los dos DNS (`172.16.6.20` y `172.16.6.21`). Los dos servidores tienen el firewall (UFW) activo con los mismos puertos abiertos.

## Tabla de servicios con su dominio

Muestra de un vistazo cada servicio, su nombre de dominio, en qué máquina está y si responde. Se crea **una vez** en `server` y se ejecuta cuando haga falta:

```bash
cat > ~/servicios.sh << 'EOF'
#!/bin/bash
P=172.16.6.20; S=172.16.6.21; Z=jordi.asir2.iespoveda.local
estado(){ [ "$1" = "ok" ] && echo "EN MARCHA" || echo "CAIDO"; }
hb(){ curl -s -m 5 -X POST -H "Content-Type: application/json" -d '{"command":"ha-heartbeat","service":["dhcp4"]}' http://$1:8001/ | grep -o '"state": "[a-z-]*"' | cut -d'"' -f4; }
web(){ [ "$(curl -s -m 5 -o /dev/null -w '%{http_code}' --resolve $1.$Z:80:$2 http://$1.$Z/)" = "200" ] && echo ok; }
dnsp=$([ "$(dig +short +time=2 +tries=1 @$P dns.$Z)" = "$P" ] && echo ok)
dnss=$([ "$(dig +short +time=2 +tries=1 @$S dns2.$Z)" = "$S" ] && echo ok)
dhp=$([ "$(hb $P)" = "hot-standby" ] && echo ok)
dhs=$([ "$(hb $S)" = "hot-standby" ] && echo ok)
{
echo "SERVICIO|DOMINIO|MAQUINA|ESTADO"
echo "DNS primario|dns.$Z|server ($P)|$(estado $dnsp)"
echo "DNS secundario|dns2.$Z|server2 ($S)|$(estado $dnss)"
echo "DHCP primario|dhcp.$Z|server ($P)|$(estado $dhp)"
echo "DHCP secundario|dhcp2.$Z|server2 ($S)|$(estado $dhs)"
echo "Web principal|www.$Z|server ($P)|$(estado "$(web www $P)")"
echo "Web secundaria|www2.$Z|server2 ($S)|$(estado "$(web www2 $S)")"
} | column -t -s'|'
EOF
chmod +x ~/servicios.sh
```

Ejecutar:

```bash
~/servicios.sh
```

Resultado esperado:

```text
SERVICIO         DOMINIO                            MAQUINA                ESTADO
DNS primario     dns.jordi.asir2.iespoveda.local    server (172.16.6.20)   EN MARCHA
DNS secundario   dns2.jordi.asir2.iespoveda.local   server2 (172.16.6.21)  EN MARCHA
DHCP primario    dhcp.jordi.asir2.iespoveda.local   server (172.16.6.20)   EN MARCHA
DHCP secundario  dhcp2.jordi.asir2.iespoveda.local  server2 (172.16.6.21)  EN MARCHA
Web principal    www.jordi.asir2.iespoveda.local    server (172.16.6.20)   EN MARCHA
Web secundaria   www2.jordi.asir2.iespoveda.local   server2 (172.16.6.21)  EN MARCHA
```

Se ejecuta desde `server` y comprueba **por red** las dos máquinas, así que no hace falta entrar en `server2`. Un servicio parado sale como `CAIDO`.

## Demostración detallada en un solo comando

Comprueba lo mismo pero con más detalle: cada nombre resuelto por **los dos** DNS, el serial de la zona y los dos Kea. Se crea **una vez** en `server`:

```bash
cat > ~/demo.sh << 'EOF'
#!/bin/bash
P=172.16.6.20; S=172.16.6.21; Z=jordi.asir2.iespoveda.local
ok(){ printf "  [ OK ]   %s\n" "$1"; }
ko(){ printf "  [FALLO]  %s\n" "$1"; }

echo "== Servicios en server ($P) =="
for s in named kea-dhcp4-server nginx; do
  systemctl is-active --quiet $s && ok "$s activo" || ko "$s NO activo"
done

echo "== DNS: nombres de server ($P), consultados a los dos DNS =="
for n in dns www dhcp; do
  for srv in $P $S; do
    r=$(dig +short +time=2 +tries=1 @$srv $n.$Z)
    [ "$r" = "$P" ] && ok "$n.$Z -> $r  (responde $srv)" || ko "$n.$Z  (responde $srv: '${r:-nada}')"
  done
done

echo "== DNS: nombres de server2 ($S), consultados a los dos DNS =="
for n in dns2 www2 dhcp2; do
  for srv in $P $S; do
    r=$(dig +short +time=2 +tries=1 @$srv $n.$Z)
    [ "$r" = "$S" ] && ok "$n.$Z -> $r  (responde $srv)" || ko "$n.$Z  (responde $srv: '${r:-nada}')"
  done
done

sp=$(dig +short @$P SOA $Z | awk '{print $3}')
ss=$(dig +short @$S SOA $Z | awk '{print $3}')
[ -n "$sp" ] && [ "$sp" = "$ss" ] && ok "Zona sincronizada: mismo serial en ambos ($sp)" || ko "Serial distinto (primario=${sp:-?} secundario=${ss:-?})"

echo "== DHCP (Kea alta disponibilidad) =="
for srv in $P $S; do
  r=$(curl -s -m 5 -X POST -H "Content-Type: application/json" -d '{"command":"ha-heartbeat","service":["dhcp4"]}' http://$srv:8001/)
  st=$(echo "$r" | grep -o '"state": "[a-z-]*"' | cut -d'"' -f4)
  [ "$st" = "hot-standby" ] && ok "Kea en $srv: $st" || ko "Kea en $srv: ${st:-sin respuesta}"
done

echo "== Webs =="
for par in "www:$P" "www2:$S"; do
  n=${par%%:*}; ip=${par##*:}
  code=$(curl -s -m 5 -o /dev/null -w '%{http_code}' --resolve $n.$Z:80:$ip http://$n.$Z/)
  [ "$code" = "200" ] && ok "http://$n.$Z -> HTTP $code" || ko "http://$n.$Z -> HTTP ${code:-sin respuesta}"
done
EOF
chmod +x ~/demo.sh
```

Ejecutar:

```bash
~/demo.sh
```

✅ Todo debe salir `[ OK ]`. Cada línea es una comprobación: servicios en marcha, los seis nombres resueltos por los dos DNS, zona sincronizada (mismo serial), los dos Kea en `hot-standby` y las dos webs respondiendo.

## Comprobaciones sueltas

Para enseñar algo concreto sin lanzar el script entero.

### Servicios activos

En `server`:

```bash
systemctl is-active named kea-dhcp4-server nginx
```

En `server2` (lleva los mismos tres):

```bash
systemctl is-active named kea-dhcp4-server nginx
```

Debe devolver `active` por cada servicio. Puertos en escucha:

```bash
sudo ss -tulpn | grep -E ':53 |:67 |:80 |:8001 '
```

### DNS

Resolver los nombres preguntando a cada servidor (`172.16.6.20` primario, `172.16.6.21` secundario):

```bash
dig @172.16.6.20 dns.jordi.asir2.iespoveda.local +short
dig @172.16.6.21 dns.jordi.asir2.iespoveda.local +short
dig @172.16.6.20 www2.jordi.asir2.iespoveda.local +short
dig @172.16.6.21 dhcp2.jordi.asir2.iespoveda.local +short
```

Zona sincronizada: los dos deben dar el **mismo serial** (tercer número):

```bash
dig @172.16.6.20 jordi.asir2.iespoveda.local SOA +short
dig @172.16.6.21 jordi.asir2.iespoveda.local SOA +short
```

Quién es la autoridad de la zona (deben salir `dns` y `dns2`), y que el secundario responde con autoridad (flag `aa`):

```bash
dig @172.16.6.20 jordi.asir2.iespoveda.local NS +short
dig @172.16.6.21 www.jordi.asir2.iespoveda.local | grep -E 'status|flags'
```

### DHCP

Estado de los dos Kea. Los dos deben decir `hot-standby`; el primario con `"scopes": ["server"]` y el secundario con `"scopes": []`:

```bash
curl -s -X POST -H "Content-Type: application/json" -d '{"command":"ha-heartbeat","service":["dhcp4"]}' http://172.16.6.20:8001/
curl -s -X POST -H "Content-Type: application/json" -d '{"command":"ha-heartbeat","service":["dhcp4"]}' http://172.16.6.21:8001/
```

Concesiones activas (deben ser las mismas en `server` y en `server2`):

```bash
sudo cat /var/lib/kea/kea-leases4.csv
```

Log en directo, para ver cómo se reparte una IP en el momento:

```bash
sudo journalctl -u kea-dhcp4-server -f
```

### Web

Web principal (en `server`) y secundaria (en `server2`). Cada una debe devolver el título de su página:

```bash
curl -sS --resolve www.jordi.asir2.iespoveda.local:80:172.16.6.20 http://www.jordi.asir2.iespoveda.local | grep '<h1>'
curl -sS --resolve www2.jordi.asir2.iespoveda.local:80:172.16.6.21 http://www2.jordi.asir2.iespoveda.local | grep '<h1>'
```

### Firewall

En cada servidor, reglas activas:

```bash
sudo ufw status numbered
```

Puertos que deben estar abiertos: `22/tcp` (SSH), `53` (DNS), `67/udp` (DHCP), `80/tcp` (web) y `8001/tcp` solo hacia el otro servidor (heartbeat de Kea). En `server` se abre desde `172.16.6.21` y en `server2` desde `172.16.6.20`.

Una prueba que **no** vale: consultar un servicio desde la propia máquina. El tráfico a sí misma va por la interfaz interna y el firewall lo deja pasar siempre. Para validar el firewall, hay que probar desde la **otra** máquina.

### Desde un cliente

Cliente Ubuntu con NetworkManager. Qué IP tiene, qué servidor DHCP se la dio y qué DNS usa:

```bash
ip -4 a
nmcli -f ALL device show | grep -i -E 'dhcp_server_identifier|ip4.dns'
```

Resolver por nombre preguntando directamente al servidor:

```bash
dig @172.16.6.20 www.jordi.asir2.iespoveda.local +short
```

Cliente Windows (PowerShell):

```powershell
ipconfig /all
nslookup www.jordi.asir2.iespoveda.local 172.16.6.20
```

En `ipconfig /all` se ve el servidor DHCP y los dos DNS recibidos.

**Atención en clientes Ubuntu:** `.local` está reservado para mDNS (DNS multidifusión), y `systemd-resolved` no envía esos nombres a un DNS normal. Aunque el cliente tenga bien puesto el DNS, `curl` o el navegador pueden dar *Could not resolve host* mientras `dig @172.16.6.20 ...` sí funciona. Se arregla indicando que este dominio concreto va por DNS (cambio temporal, se pierde al reiniciar la red):

```bash
IF=$(ip route show default | awk '{print $5}' | head -1)
sudo resolvectl domain $IF '~jordi.asir2.iespoveda.local'
curl -sI http://www.jordi.asir2.iespoveda.local | head -1
```

Debe salir `HTTP/1.1 200 OK`.

## Demostrar la redundancia del DNS

Con el primario parado, el secundario sigue respondiendo.

**1.** En `server`, parar el DNS:

```bash
sudo systemctl stop named
```

**2.** Desde cualquier equipo, preguntar al secundario:

```bash
dig @172.16.6.21 www.jordi.asir2.iespoveda.local +short
```

✅ Debe seguir devolviendo `172.16.6.20`.

**3.** **Volver a arrancar el primario** (no olvidarlo):

```bash
sudo systemctl start named
```

## Si algo sale mal

| Síntoma | Qué mirar |
|---|---|
| Un servicio sale `inactive` | `sudo systemctl start <servicio>` y `journalctl -u <servicio> -e` |
| El secundario devuelve otro serial | `sudo rndc retransfer jordi.asir2.iespoveda.local` en `server2` y repetir el `dig` |
| Kea no sale `hot-standby` | Esperar 15 s; `ready` es un estado de paso tras reiniciar. Si sigue igual, comprobar que el otro Kea está activo y que el puerto `8001/tcp` está permitido en `ufw` |
| `curl` al `8001` da timeout | Falta la regla del firewall hacia el otro servidor (ver `Unit2_DHCP_server.md`, apartado 7) |
| El cliente no resuelve el nombre | `.local` va por mDNS en Ubuntu: usar `resolvectl domain` (ver "Desde un cliente") o probar con `dig @172.16.6.20 <nombre>` |
| `ssh` avisa de *host identification has changed* tras cambiar la IP de una máquina | La IP antes era de otra máquina: `ssh-keygen -R <ip>` en el equipo cliente y volver a conectar |
