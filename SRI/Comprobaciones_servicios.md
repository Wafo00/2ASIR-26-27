# Comprobaciones rápidas: qué servicios hay, dónde están y si funcionan

> Para cuando alguien pregunte: *¿qué servicios tienes activos, dónde, funcionan, me lo demuestras?* Un resumen, un script que lo comprueba todo, y los comandos sueltos por si hay que enseñar algo concreto.

## Índice

- [Qué hay montado](#qué-hay-montado)
- [Demostración en un solo comando](#demostración-en-un-solo-comando)
- [Comprobaciones sueltas](#comprobaciones-sueltas)
  - [Servicios activos](#servicios-activos)
  - [DNS](#dns)
  - [DHCP](#dhcp)
  - [Web](#web)
  - [Desde un cliente](#desde-un-cliente)
- [Demostrar la redundancia del DNS](#demostrar-la-redundancia-del-dns)
- [Si algo sale mal](#si-algo-sale-mal)

## Qué hay montado

| Servicio | Programa | Dónde | Nombre |
|---|---|---|---|
| DNS primario | BIND9 | `server` (`172.16.6.20`) | `dns.jordi.asir2.iespoveda.local` |
| DNS secundario | BIND9 | `server2` (`172.16.6.23`) | recibe las zonas del primario |
| DHCP primario | Kea (HA, hot-standby) | `server` (`172.16.6.20`) | `dhcp.jordi.asir2.iespoveda.local` |
| DHCP secundario | Kea (HA, en espera) | `server2` (`172.16.6.23`) | sincronizado con el primario |
| Web | Nginx | `server` (`172.16.6.20`) | `www.jordi.asir2.iespoveda.local` |

Red `172.16.6.0/24`, puerta de enlace `172.16.6.1`, pool DHCP `172.16.6.200 - 172.16.6.250`.

## Demostración en un solo comando

Se crea **una vez** en `server` y se ejecuta cuando haga falta:

```bash
cat > ~/demo.sh << 'EOF'
#!/bin/bash
P=172.16.6.20; S=172.16.6.23; Z=jordi.asir2.iespoveda.local
ok(){ printf "  [ OK ]   %s\n" "$1"; }
ko(){ printf "  [FALLO]  %s\n" "$1"; }

echo "== Servicios en server ($P) =="
for s in named kea-dhcp4-server nginx; do
  systemctl is-active --quiet $s && ok "$s activo" || ko "$s NO activo"
done

echo "== DNS (primario $P y secundario $S) =="
for n in dns www dhcp; do
  for srv in $P $S; do
    r=$(dig +short +time=2 +tries=1 @$srv $n.$Z)
    [ "$r" = "$P" ] && ok "$n.$Z -> $r  (responde $srv)" || ko "$n.$Z  (responde $srv: '${r:-nada}')"
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

echo "== Web =="
code=$(curl -s -m 5 -o /dev/null -w '%{http_code}' --resolve www.$Z:80:$P http://www.$Z/)
[ "$code" = "200" ] && ok "http://www.$Z -> HTTP $code" || ko "http://www.$Z -> HTTP ${code:-sin respuesta}"
EOF
chmod +x ~/demo.sh
```

Ejecutar:

```bash
~/demo.sh
```

✅ Todo debe salir `[ OK ]`. Cada línea es una comprobación: servicios en marcha, los tres nombres resueltos por **los dos** DNS, zona sincronizada (mismo serial), los dos Kea en `hot-standby` y la web respondiendo.

## Comprobaciones sueltas

Para enseñar algo concreto sin lanzar el script entero.

### Servicios activos

En `server`:

```bash
systemctl is-active named kea-dhcp4-server nginx
```

En `server2`:

```bash
systemctl is-active named kea-dhcp4-server
```

Debe devolver `active` por cada servicio. Puertos en escucha:

```bash
sudo ss -tulpn | grep -E ':53 |:67 |:80 |:8001 '
```

### DNS

Resolver los tres nombres preguntando a cada servidor (`172.16.6.20` primario, `172.16.6.23` secundario):

```bash
dig @172.16.6.20 dns.jordi.asir2.iespoveda.local +short
dig @172.16.6.23 dns.jordi.asir2.iespoveda.local +short
dig @172.16.6.20 www.jordi.asir2.iespoveda.local +short
dig @172.16.6.23 dhcp.jordi.asir2.iespoveda.local +short
```

Zona sincronizada: los dos deben dar el **mismo serial** (tercer número):

```bash
dig @172.16.6.20 jordi.asir2.iespoveda.local SOA +short
dig @172.16.6.23 jordi.asir2.iespoveda.local SOA +short
```

Quién es la autoridad de la zona, y que el secundario responde con autoridad (flag `aa`):

```bash
dig @172.16.6.20 jordi.asir2.iespoveda.local NS +short
dig @172.16.6.23 www.jordi.asir2.iespoveda.local | grep -E 'status|flags'
```

### DHCP

Estado de los dos Kea. Los dos deben decir `hot-standby`; el primario con `"scopes": ["server"]` y el secundario con `"scopes": []`:

```bash
curl -s -X POST -H "Content-Type: application/json" -d '{"command":"ha-heartbeat","service":["dhcp4"]}' http://172.16.6.20:8001/
curl -s -X POST -H "Content-Type: application/json" -d '{"command":"ha-heartbeat","service":["dhcp4"]}' http://172.16.6.23:8001/
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

```bash
curl -sS --resolve www.jordi.asir2.iespoveda.local:80:172.16.6.20 http://www.jordi.asir2.iespoveda.local | grep '<h1>'
```

Debe devolver el título de la página.

### Desde un cliente

Cliente Ubuntu con NetworkManager. Qué IP tiene, qué servidor DHCP se la dio y qué DNS usa:

```bash
ip -4 a
nmcli -f ALL device show | grep -i -E 'dhcp_server_identifier|ip4.dns'
```

Resolver por nombre (preguntando al servidor, que es lo que cuenta aunque `.local` use mDNS):

```bash
dig @172.16.6.20 www.jordi.asir2.iespoveda.local +short
```

Cliente Windows (PowerShell):

```powershell
ipconfig /all
nslookup www.jordi.asir2.iespoveda.local 172.16.6.20
```

En `ipconfig /all` se ve el servidor DHCP y los dos DNS recibidos.

## Demostrar la redundancia del DNS

Con el primario parado, el secundario sigue respondiendo.

**1.** En `server`, parar el DNS:

```bash
sudo systemctl stop named
```

**2.** Desde cualquier equipo, preguntar al secundario:

```bash
dig @172.16.6.23 www.jordi.asir2.iespoveda.local +short
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
| Kea no sale `hot-standby` | Esperar 15 s; si sigue igual, comprobar que el otro Kea está activo y que el puerto `8001/tcp` está permitido en `ufw` |
| `curl` al `8001` da timeout | El servidor tiene varias IPs o falta la regla del firewall (ver `Unit2_DHCP_server.md`, apartado 7) |
| El cliente no resuelve el nombre | `.local` puede ir por mDNS: probar con `dig @172.16.6.20 <nombre>` |
