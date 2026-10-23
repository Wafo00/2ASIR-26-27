# Instalación y configuración de DNS con BIND9 (Ubuntu)

> Antes de instalar: comprobar que no hay otro servicio DNS escuchando en el puerto 53 de la IP del servidor.
>
> ```
> sudo ss -tulpn | grep ':53'
> ```
>
> En Ubuntu, `systemd-resolved` escucha solo en `127.0.0.53`; no entra en conflicto con BIND si este escucha en otras direcciones.

## Índice

- [Cómo se forman los nombres de dominio](#cómo-se-forman-los-nombres-de-dominio)
- [Conceptos](#conceptos)
- [1. Instalar](#1-instalar)
- [2. Opciones generales y reenvío](#2-opciones-generales-y-reenvío)
- [3. Declarar las zonas](#3-declarar-las-zonas)
- [4. Crear los ficheros de zona](#4-crear-los-ficheros-de-zona)
- [5. Validar](#5-validar)
- [6. Arrancar y comprobar](#6-arrancar-y-comprobar)
- [7. Pruebas](#7-pruebas)
  - [Probar desde un cliente](#probar-desde-un-cliente)
- [8. Configurar los clientes](#8-configurar-los-clientes)
- [9. Firewall (opcional)](#9-firewall-opcional)
- [10. Zona con un nombre por servicio y webs de prueba](#10-zona-con-un-nombre-por-servicio-y-webs-de-prueba)
  - [10.1 Crear la zona](#101-crear-la-zona)
  - [10.2 Webs de prueba con Nginx (opcional)](#102-webs-de-prueba-con-nginx-opcional)
  - [10.3 Demostrar que cada servicio tiene su nombre](#103-demostrar-que-cada-servicio-tiene-su-nombre)
- [11. Servidor secundario (esclavo) y transferencia de zonas](#11-servidor-secundario-esclavo-y-transferencia-de-zonas)
  - [11.1 En el primario](#111-en-el-primario)
  - [11.2 En la máquina del secundario](#112-en-la-máquina-del-secundario)
  - [11.3 Comprobar la transferencia](#113-comprobar-la-transferencia)
  - [11.4 Probar la redundancia](#114-probar-la-redundancia)
  - [11.5 Que los clientes usen los dos servidores](#115-que-los-clientes-usen-los-dos-servidores)
  - [11.6 Hacer un cambio con el secundario montado](#116-hacer-un-cambio-con-el-secundario-montado)
  - [11.7 Añadir una zona nueva a un secundario que ya funciona](#117-añadir-una-zona-nueva-a-un-secundario-que-ya-funciona)
- [Operaciones habituales](#operaciones-habituales)
  - [Cambiar la IP o la red del servidor](#cambiar-la-ip-o-la-red-del-servidor)
  - [Recuperar named tras un allow-transfer duplicado](#recuperar-named-tras-un-allow-transfer-duplicado)

## Cómo se forman los nombres de dominio

Un nombre de dominio se lee **de derecha a izquierda**, de lo más general a lo más concreto. Cada tramo separado por puntos es una *etiqueta* y cada etiqueta es un *nivel*:

```
.                  ← raíz (nivel 0)
└── es             ← TLD, dominio de primer nivel (nivel 1)
    └── ejemplo    ← dominio de segundo nivel (nivel 2)
        └── www    ← host o subdominio (nivel 3)
```

Se lee `www.ejemplo.es.`: la raíz delega en `es`, `es` delega en `ejemplo`, y `ejemplo` define el host `www`. Cada nivel indica quién es el siguiente mediante registros `NS`. La raíz la sirven 13 identidades de servidores (letras A a M), replicadas en cientos de instancias por todo el mundo.

| Nivel | Qué es | Ejemplo |
|---|---|---|
| Raíz | Origen de todo el árbol; es el punto final del nombre | `.` |
| Dominio de primer nivel (TLD) | Genérico, de país o especial | `com`, `org`, `es`, `local` |
| Dominio de segundo nivel | El que registra una organización | `google.com`, `ASIR2.local` |
| Subdominio / host | Lo que cada organización crea dentro de su dominio | `www.google.com`, `equipo1.ASIR2.local` |

| Tipo de TLD | Qué es | Ejemplos |
|---|---|---|
| Genérico (gTLD) | Uso general, sin país asociado | `com`, `org`, `net`, `info`, `biz` |
| Restringido o patrocinado | Reservado a un tipo de entidad | `edu`, `gov`, `mil`, `int` |
| De país (ccTLD) | Dos letras (ISO 3166) | `es`, `fr`, `de`, `uk` |
| Nuevos gTLD | Temas, marcas o ciudades | `app`, `dev`, `madrid` |
| Infraestructura | Para el funcionamiento del propio DNS | `arpa` (de ahí `in-addr.arpa`, las zonas inversas) |
| Uso especial o privado | No existen en el DNS público | `local`, `internal`, `home.arpa`, `localhost`, `test`, `example`, `invalid` |

**A nivel global (DNS público):** hay un único árbol jerárquico y su gestión está **delegada**. La raíz la coordinan IANA/ICANN, cada TLD lo gestiona un registro, y las organizaciones registran sus dominios de segundo nivel a través de un registrador. Para resolver `www.google.com`, un servidor pregunta a la raíz, esta lo remite al servidor de `com`, y este al de `google.com`.

**A nivel local (red privada):** la organización monta su propio servidor DNS y define una zona propia que solo se resuelve dentro de su red. No hace falta registrarla, pero no debe coincidir con un dominio público que se quiera seguir usando. Sufijos pensados para esto:

| Sufijo | Uso |
|---|---|
| `.internal` | Reservado para uso privado. |
| `.home.arpa` | Reservado para redes domésticas (RFC 8375). |
| `.local` | Reservado para mDNS (RFC 6762). En esta práctica se usa `ASIR2.local.` |

Reglas de los nombres:

- Solo letras, números y guiones; sin espacios, y sin guion al principio ni al final de una etiqueta.
- No distinguen mayúsculas de minúsculas (`ASIR2.local` = `asir2.local`).
- Cada etiqueta admite como máximo 63 caracteres, y el nombre completo 253.
- Un nombre que **termina en punto** es absoluto (*FQDN*): `equipo1.ASIR2.local.`. En los ficheros de zona, sin el punto final BIND añade el nombre de la zona y queda duplicado.
- Nombres de equipo de la práctica: `equipo1`, `equipo2`... (no se admiten espacios).

## Conceptos

| Concepto | Qué es |
|---|---|
| Servidor autoritativo | Responde con datos de las zonas que gestiona él mismo (respuesta con flag `aa`). |
| Servidor recursivo | Busca respuestas de dominios que no son suyos en nombre del cliente (flag `ra` = recursión disponible). |
| Forwarder (reenviador) | Servidor al que se reenvían las consultas que no son de zonas propias. Aquí, Google (`8.8.8.8`, `8.8.4.4`). |
| Zona | Parte del espacio de nombres que este servidor gestiona (`ASIR2.local`). |
| Zona directa | Traduce nombre → IP. Usa registros `A`. |
| Zona inversa | Traduce IP → nombre. Usa registros `PTR`. Su nombre es la red con los octetos al revés + `.in-addr.arpa` (`172.16.6.0/24` → `6.16.172.in-addr.arpa`). |
| SOA | Primer registro de toda zona: servidor primario, contacto del administrador (el primer punto equivale a `@`), serial y temporizadores. |
| Serial | Versión de la zona. Se incrementa en cada cambio (convención `AAAAMMDDNN`). |
| NS | Indica qué servidor es la autoridad de la zona. |
| A / PTR / CNAME | Nombre → IPv4 / IP → nombre / alias de otro nombre. |
| TTL | Segundos que otros pueden guardar en caché una respuesta. |
| FQDN y punto final | Un nombre completo termina en punto (`equipo1.ASIR2.local.`). Sin punto, BIND le añade el nombre de la zona y queda duplicado. |
| Primario y secundario | El primario (antes *maestro*) guarda la zona original y se edita ahí. El secundario (antes *esclavo*) guarda una copia que obtiene por transferencia. Ambos responden con autoridad (`aa`). |
| Transferencia de zona | Copia de la zona del primario al secundario. `AXFR` copia la zona entera; `IXFR` solo los cambios. Va por TCP, puerto 53. |
| NOTIFY | Aviso del primario al secundario de que la zona ha cambiado, para que la pida sin esperar. |
| Refresh / Retry / Expire (SOA) | Cada cuántos segundos el secundario comprueba si hay cambios (Refresh), cada cuántos reintenta si falla (Retry), y a los cuántos sin contactar con el primario deja de responder (Expire). |

## 1. Instalar

```
sudo apt update
sudo apt install bind9 bind9-utils bind9-dnsutils
```

`bind9-utils` aporta los comandos `named-checkconf` y `named-checkzone`; `bind9-dnsutils` aporta `dig` y `nslookup`. El servicio se llama `named` (`bind9` es un alias).

## 2. Opciones generales y reenvío

```
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
sudo tee /etc/bind/named.conf.options > /dev/null << 'EOF'
options {
    directory "/var/cache/bind";

    // IPs por las que escucha
    listen-on { 127.0.0.1; 172.16.6.20; };
    listen-on-v6 { none; };

    // Quién puede consultar y quién puede usar recursión
    allow-query     { localhost; 172.16.6.0/24; };
    allow-recursion { localhost; 172.16.6.0/24; };

    // Lo que no sea de zonas propias se reenvía a Google
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;

    dnssec-validation auto;
};
EOF
```

Sustituir `172.16.6.20` y `172.16.6.0/24` por la IP y la red del servidor.

| Opción | Para qué sirve |
|---|---|
| `listen-on` | IPs del servidor donde atiende peticiones. Si la IP del servidor no figura aquí, no responde en ella. |
| `allow-query` / `allow-recursion` | Limitan quién consulta y quién puede pedir resolución externa. Sin límite, el servidor sería un *resolver abierto* (abusable desde internet). |
| `forwarders` | Servidores a los que se reenvía lo que no es de zonas propias. |
| `forward only` | Solo usa los forwarders. Con `forward first`, si fallan, intentaría resolver por su cuenta. |
| `dnssec-validation auto` | Valida firmas DNSSEC de las respuestas externas. |

## 3. Declarar las zonas

```
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
sudo tee -a /etc/bind/named.conf.local > /dev/null << 'EOF'

zone "ASIR2.local" {
    type primary;
    file "/etc/bind/zones/db.ASIR2.local";
};

zone "6.16.172.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.172.16.6";
};
EOF
```

`type primary` indica que este servidor guarda la zona original (no una copia de otro). En `named.conf.local` el nombre de la zona va **sin** punto final.

## 4. Crear los ficheros de zona

```
sudo mkdir -p /etc/bind/zones
```

**Zona directa** (nombre → IP):

```
sudo tee /etc/bind/zones/db.ASIR2.local > /dev/null << 'EOF'
$TTL 3600
@           IN  SOA ServerDNS20.ASIR2.local. admin.ASIR2.local. (
                    2026101301  ; Serial (AAAAMMDDNN)
                    3600        ; Refresh
                    900         ; Retry
                    604800      ; Expire
                    3600 )      ; Negative cache TTL

@           IN  NS  ServerDNS20.ASIR2.local.

ServerDNS20 IN  A   172.16.6.20
equipo1     IN  A   172.16.6.21
equipo2     IN  A   172.16.6.22
equipo3     IN  A   172.16.6.23
equipo4     IN  A   172.16.6.24
equipo5     IN  A   172.16.6.25
equipo6     IN  A   172.16.6.26
equipo7     IN  A   172.16.6.27
equipo8     IN  A   172.16.6.28
EOF
```

**Zona inversa** (IP → nombre). Solo se escribe el último octeto; el resto lo da el nombre de la zona:

```
sudo tee /etc/bind/zones/db.172.16.6 > /dev/null << 'EOF'
$TTL 3600
@   IN  SOA ServerDNS20.ASIR2.local. admin.ASIR2.local. (
            2026101301  ; Serial (AAAAMMDDNN)
            3600        ; Refresh
            900         ; Retry
            604800      ; Expire
            3600 )      ; Negative cache TTL

@   IN  NS  ServerDNS20.ASIR2.local.

20  IN  PTR ServerDNS20.ASIR2.local.
21  IN  PTR equipo1.ASIR2.local.
22  IN  PTR equipo2.ASIR2.local.
23  IN  PTR equipo3.ASIR2.local.
24  IN  PTR equipo4.ASIR2.local.
25  IN  PTR equipo5.ASIR2.local.
26  IN  PTR equipo6.ASIR2.local.
27  IN  PTR equipo7.ASIR2.local.
28  IN  PTR equipo8.ASIR2.local.
EOF
```

Nota: `ServerDNS20` es el nombre del servidor **dentro del DNS**. Es independiente del nombre de la máquina (`hostname`), que puede ser distinto.

## 5. Validar

```
sudo named-checkconf
sudo named-checkzone ASIR2.local /etc/bind/zones/db.ASIR2.local
sudo named-checkzone 6.16.172.in-addr.arpa /etc/bind/zones/db.172.16.6
```

`named-checkconf` no muestra nada si todo está bien. `named-checkzone` debe terminar en `OK`.

## 6. Arrancar y comprobar

```
sudo systemctl restart named
sudo systemctl status named
systemctl is-enabled named
```

Debe salir `active (running)` y `enabled` (viene habilitado por defecto). Si falla: `journalctl -u named -e`

Comprobar que ha cargado las dos zonas y en qué IPs escucha:

```
journalctl -u named -e | grep -i "loaded serial"
sudo ss -tulpn | grep ':53' | grep named
```

Deben aparecer `ASIR2.local` y `6.16.172.in-addr.arpa`, y `named` escuchando en `127.0.0.1` y en la IP del servidor.

## 7. Pruebas

```
dig @172.16.6.20 equipo1.ASIR2.local +short
dig @172.16.6.20 -x 172.16.6.21 +short
dig @172.16.6.20 ServerDNS20.ASIR2.local +short
dig @172.16.6.20 google.com +short
```

Resultados esperados:

- Primera consulta: `172.16.6.21` (zona directa).
- Segunda: `equipo1.ASIR2.local.` (zona inversa).
- Tercera: `172.16.6.20` (el propio servidor).
- Cuarta: una o varias IPs de Google (reenvío).

Comprobar los 8 equipos de una vez:

```
for i in $(seq 1 8); do dig @172.16.6.20 equipo$i.ASIR2.local +short; done
```

Distinguir respuesta propia de reenviada:

```
dig @172.16.6.20 equipo1.ASIR2.local | grep -E 'status|flags'
dig @172.16.6.20 google.com | grep -E 'status|flags'
```

En la primera aparece `aa` (autoritativa); en la segunda no (viene del forwarder). Ambas deben dar `status: NOERROR`.

Desde un cliente Windows:

```
nslookup equipo1.ASIR2.local 172.16.6.20
```

> **Nota sobre `.local`:** en clientes Linux con `avahi`/`nss-mdns`, los nombres `.local` se intentan resolver por mDNS. Puede ocurrir que `dig` y `nslookup` funcionen y que `ping equipo1.ASIR2.local` falle. Para comprobar el DNS, usar siempre `dig` o `nslookup`.

### Probar desde un cliente

El cliente debe estar en la **misma red** que el servidor; entre redes distintas, `dig` da *timeout*. Configuración rápida de IP fija y DNS en un cliente Ubuntu (sin puerta de enlace, que para esta prueba no hace falta). Antes, copia de seguridad del perfil:

```
sudo cp /etc/netplan/90-NM-*.yaml ~/90-NM.yaml.orig
CON=$(nmcli -t -f NAME,DEVICE connection show --active | grep ':enp0s3$' | cut -d: -f1)
sudo nmcli connection modify "$CON" ipv4.method manual ipv4.addresses 172.16.6.40/24 ipv4.gateway "" ipv4.dns 172.16.6.20 ipv4.dns-search ASIR2.local ipv4.ignore-auto-dns yes
sudo nmcli connection up "$CON"
resolvectl status | grep -i 'dns server'
dig equipo1.ASIR2.local +short
```

Con el DNS configurado, `dig` sin `@` también funciona. Para volver a la configuración anterior:

```
sudo cp ~/90-NM.yaml.orig /etc/netplan/90-NM-*.yaml
sudo netplan apply
```

> Los ficheros `.bak` de `/etc/netplan` los ignora netplan: editarlos no tiene efecto. Ver cuáles están activos con `ls /etc/netplan/`.

## 8. Configurar los clientes

**Por DHCP (Kea)**, en `option-data` de la subred:

```
{ "name": "domain-name-servers", "data": "172.16.6.20" },
{ "name": "domain-name", "data": "ASIR2.local" }
```

**Por DHCP (Windows Server):**

```
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer 172.16.6.20 -DnsDomain ASIR2.local
```

**Cliente Ubuntu con IP fija (netplan):**

```
nameservers:
  addresses: [172.16.6.20]
  search: [ASIR2.local]
```

Los registros DNS son fijos, así que cada equipo debe conservar siempre su IP. Reservarla por MAC en el DHCP (Kea, dentro de `subnet4`):

```
"reservations": [
  { "hw-address": "08:00:27:aa:bb:01", "ip-address": "172.16.6.21", "hostname": "equipo1" }
]
```

(una entrada por equipo).

## 9. Firewall (opcional)

```
sudo ufw status
```

Si está `inactive`, no hace falta abrir nada. Si se va a activar, abrir **antes** el puerto de SSH o se pierde el acceso remoto:

```
sudo ufw allow 22/tcp
sudo ufw allow 53
sudo ufw allow 67/udp
sudo ufw enable
sudo ufw status verbose
```

`53`: DNS (TCP y UDP). `67/udp`: DHCP, solo si el servidor también reparte IPs. Añadir también los puertos de otros servicios del servidor (por ejemplo `80/tcp` y `8080/tcp` para web). Antes de cerrar la sesión SSH actual, comprobar que una sesión nueva entra.

## 10. Zona con un nombre por servicio y webs de prueba

Una misma máquina puede dar varios servicios. Para saber **qué es cada cosa**, cada servicio tiene su propio nombre, que sigue la jerarquía de la red: `servicio . alumno . módulo . centro . local`. Aquí se crea la zona `jordi.asir2.iespoveda.local` con los nombres de los servicios de la máquina.

| Nombre | Qué es | Apunta a |
|---|---|---|
| `dns.jordi.asir2.iespoveda.local` | Servidor DNS de la zona | `172.16.6.20` |
| `www.jordi.asir2.iespoveda.local` | Web | `172.16.6.20` |
| `dhcp.jordi.asir2.iespoveda.local` | Servidor DHCP (Kea) | `172.16.6.20` |

Para otro servicio (`ftp`, `mail`...) basta una línea `A` más en la zona.

> `.local` está reservado para mDNS: algunos clientes (Linux con `avahi`, macOS) no preguntan al DNS por nombres `.local`. Si un cliente no resuelve, probar con `dig @172.16.6.20 <nombre>`, que sí consulta al servidor.

### 10.1 Crear la zona

Declarar la zona **debajo** de las existentes (la primera línea evita duplicarla si se ejecuta dos veces) y crear el fichero con los nombres:

```
grep -q 'zone "jordi.asir2.iespoveda.local"' /etc/bind/named.conf.local || sudo tee -a /etc/bind/named.conf.local > /dev/null << 'EOF'

zone "jordi.asir2.iespoveda.local" {
    type primary;
    file "/etc/bind/zones/db.jordi.asir2.iespoveda.local";
};
EOF

sudo tee /etc/bind/zones/db.jordi.asir2.iespoveda.local > /dev/null << 'EOF'
$TTL 300
@      IN  SOA dns.jordi.asir2.iespoveda.local. admin.jordi.asir2.iespoveda.local. (
              2026102301 604800 86400 2419200 604800 )
@      IN  NS  dns.jordi.asir2.iespoveda.local.
dns    IN  A   172.16.6.20
www    IN  A   172.16.6.20
dhcp   IN  A   172.16.6.20
EOF

sudo named-checkconf && sudo named-checkzone jordi.asir2.iespoveda.local /etc/bind/zones/db.jordi.asir2.iespoveda.local
sudo rndc reload
```

Cada línea `nombre IN A ip` es un dominio. Los nombres se escriben **sin** el sufijo de la zona: BIND le añade `.jordi.asir2.iespoveda.local`. Para añadir otro: una línea nueva, **subir el serial**, validar y `sudo rndc reload`.

Comprobar:

```
dig @172.16.6.20 dns.jordi.asir2.iespoveda.local +short
dig @172.16.6.20 www.jordi.asir2.iespoveda.local +short
dig @172.16.6.20 jordi.asir2.iespoveda.local NS +short
```

✅ Debe salir `172.16.6.20`, `172.16.6.20` y `dns.jordi.asir2.iespoveda.local.`.

### 10.2 Web de prueba con Nginx (opcional)

El DNS no conoce puertos: una web se distingue por su nombre y su puerto.

```
sudo apt install -y nginx
sudo mkdir -p /var/www/dominioa
echo '<h1>www.jordi.asir2.iespoveda.local</h1>' | sudo tee /var/www/dominioa/index.html > /dev/null

sudo tee /etc/nginx/sites-available/dominioa > /dev/null << 'EOF'
server {
    listen 80;
    server_name www.jordi.asir2.iespoveda.local;
    root /var/www/dominioa;
    index index.html;
}
EOF

sudo ln -sf /etc/nginx/sites-available/dominioa /etc/nginx/sites-enabled/dominioa
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Si `ufw` está activo, abrir el puerto: `sudo ufw allow 80/tcp`. Para una segunda web, otro `server` con `listen 8080;` y su propio directorio.

### 10.3 Demostrar que cada servicio tiene su nombre

En el servidor (sin depender de su DNS del sistema):

```
dig @172.16.6.20 jordi.asir2.iespoveda.local NS +short
for n in dns www dhcp; do printf "%-36s %s\n" "$n.jordi.asir2.iespoveda.local" "$(dig @172.16.6.20 $n.jordi.asir2.iespoveda.local +short)"; done
curl -sS --resolve www.jordi.asir2.iespoveda.local:80:172.16.6.20 http://www.jordi.asir2.iespoveda.local | grep '<h1>'
```

Debe salir `dns.jordi.asir2.iespoveda.local.` como servidor de la zona, los tres nombres con `172.16.6.20` y el título de la web.

Desde un cliente Windows (PowerShell como administrador; `Ethernet` es el nombre del adaptador):

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 172.16.6.20
Clear-DnsClientCache
nslookup www.jordi.asir2.iespoveda.local
Start-Process "http://www.jordi.asir2.iespoveda.local"
```

Para devolver el DNS a su valor anterior:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
```

## 11. Servidor secundario (esclavo) y transferencia de zonas

Un secundario da redundancia: si el primario cae, sigue respondiendo con la copia de las zonas. **Las zonas solo se editan en el primario**; el secundario las recibe por transferencia, siempre que el **serial suba** en cada cambio.

Valores de ejemplo: primario `172.16.6.20`; secundario `ServerDNS30` en `172.16.6.30`. Sustituirlos por los reales. El secundario debe ser otra máquina, con una IP propia dentro de la red y que no coincida con ningún equipo.

### 11.1 En el primario

Autorizar la transferencia solo al secundario (sin esto, BIND la permite a cualquiera) y avisarle de los cambios. Ejecutar **una sola vez**:

```
sudo sed -i 's|type primary;|type primary;\n    allow-transfer { 172.16.6.30; };\n    also-notify { 172.16.6.30; };|' /etc/bind/named.conf.local
```

Comprobar que no ha quedado duplicado. Debe devolver **una línea por zona**: `2` (directa e inversa) o `3` si ya se creó la zona `jordi.asir2.iespoveda.local` del apartado 10:

```
grep -c 'allow-transfer' /etc/bind/named.conf.local
```

Si devuelve más que el número de zonas, el comando se ejecutó más de una vez y `named` no arrancará (`'allow-transfer' redefined`): ver la tabla de síntomas.

Registrar al secundario en las zonas (`NS`, `A` y `PTR`) y subir el serial (formato `AAAAMMDDNN`, siempre mayor que el anterior). También **una sola vez**:

```
sudo sed -i '/IN  NS  /a @           IN  NS  ServerDNS30.ASIR2.local.' /etc/bind/zones/db.ASIR2.local
sudo sed -i '/IN  NS  /a @   IN  NS  ServerDNS30.ASIR2.local.' /etc/bind/zones/db.172.16.6
echo 'ServerDNS30 IN  A   172.16.6.30' | sudo tee -a /etc/bind/zones/db.ASIR2.local > /dev/null
echo '30  IN  PTR ServerDNS30.ASIR2.local.' | sudo tee -a /etc/bind/zones/db.172.16.6 > /dev/null
sudo sed -i -E 's/[0-9]{10}( +; Serial)/2026101601\1/' /etc/bind/zones/db.ASIR2.local /etc/bind/zones/db.172.16.6
```

Validar y aplicar:

```
sudo named-checkconf
sudo named-checkzone ASIR2.local /etc/bind/zones/db.ASIR2.local
sudo named-checkzone 6.16.172.in-addr.arpa /etc/bind/zones/db.172.16.6
sudo rndc reload
```

Si `ufw` está activo, permitir al secundario (las transferencias usan TCP 53):

```
sudo ufw allow from 172.16.6.30 to any port 53
```

### 11.2 En la máquina del secundario

```
sudo apt update
sudo apt install bind9 bind9-utils bind9-dnsutils
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
sudo tee /etc/bind/named.conf.options > /dev/null << 'EOF'
options {
    directory "/var/cache/bind";

    listen-on { 127.0.0.1; 172.16.6.30; };
    listen-on-v6 { none; };

    allow-query     { localhost; 172.16.6.0/24; };
    allow-recursion { localhost; 172.16.6.0/24; };

    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;

    dnssec-validation auto;
};
EOF
```

Declarar las zonas como secundarias (la tercera, `jordi.asir2.iespoveda.local`, solo si se creó en el apartado 10):

```
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
sudo tee -a /etc/bind/named.conf.local > /dev/null << 'EOF'

zone "ASIR2.local" {
    type secondary;
    primaries { 172.16.6.20; };
    file "/var/cache/bind/db.ASIR2.local";
};

zone "6.16.172.in-addr.arpa" {
    type secondary;
    primaries { 172.16.6.20; };
    file "/var/cache/bind/db.172.16.6";
};

zone "jordi.asir2.iespoveda.local" {
    type secondary;
    primaries { 172.16.6.20; };
    file "/var/cache/bind/db.jordi.asir2.iespoveda.local";
};
EOF
sudo named-checkconf
sudo systemctl restart named
```

- **`type secondary` y `primaries`:** indican que la zona no es propia y de dónde se copia. Son los nombres actuales de `slave` y `masters`.
- **Ficheros de zona:** no hay que crearlos; se generan solos con la transferencia. Se guardan en `/var/cache/bind`, no en `/etc/bind`, porque AppArmor solo deja escribir al servicio en el primero.

> Si la máquina del secundario tiene **varias IPs**, el primario ve la IP principal de la interfaz, que puede no ser la autorizada en `allow-transfer`. Fijar la IP de origen en `options` del secundario, una sola vez: `transfer-source 172.16.6.30;`.

### 11.3 Comprobar la transferencia

Desde el secundario, antes de nada, comprobar que llega al primario: `ping -c 2 172.16.6.20`. Después:

```
journalctl -u named -e --no-pager | grep -i -E "transfer|loaded serial"
ls -l /var/cache/bind/
dig @172.16.6.30 ASIR2.local SOA +short
dig @172.16.6.20 ASIR2.local SOA +short
dig @172.16.6.30 jordi.asir2.iespoveda.local SOA +short
dig @172.16.6.20 jordi.asir2.iespoveda.local SOA +short
```

En el log debe aparecer `Transfer status: success`, en `ls` los ficheros de zona (uno por zona), y los dos `SOA` deben dar **el mismo serial**. Probar que el secundario responde con autoridad:

```
dig @172.16.6.30 equipo1.ASIR2.local | grep -E 'status|flags'
dig @172.16.6.30 -x 172.16.6.21 +short
```

Debe salir `NOERROR`, el flag `aa`, y la IP `172.16.6.21` / el nombre `equipo1.ASIR2.local.`.

Ver la transferencia completa de la zona, indicando la IP de origen autorizada (solo funciona desde el secundario; desde cualquier otro equipo da `Transfer failed`):

```
dig -b 172.16.6.30 @172.16.6.20 ASIR2.local AXFR
```

Forzar una transferencia y ver el estado de la zona en el secundario:

```
sudo rndc retransfer ASIR2.local
sudo rndc zonestatus ASIR2.local
```

Hasta la primera transferencia, `zonestatus` responde `zone not loaded`: es normal.

### 11.4 Probar la redundancia

```
sudo systemctl stop named
```

Ejecutarlo **en el primario**. Desde un cliente, `dig @172.16.6.30 equipo1.ASIR2.local +short` debe seguir respondiendo. Para recuperar el servicio: `sudo systemctl start named`. Si el primario estuviera caído más tiempo que el `Expire` del `SOA`, el secundario dejaría de responder.

### 11.5 Que los clientes usen los dos servidores

**Por DHCP (Kea):**

```
{ "name": "domain-name-servers", "data": "172.16.6.20, 172.16.6.30" }
```

**Por DHCP (Windows Server):**

```
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer 172.16.6.20,172.16.6.30
```

**Cliente Ubuntu con IP fija (netplan):**

```
nameservers:
  addresses: [172.16.6.20, 172.16.6.30]
```

El cliente pregunta al primero y, si no responde, al segundo.

### 11.6 Hacer un cambio con el secundario montado

Editar solo el primario, **subir el serial** en el fichero cambiado, validar y `sudo rndc reload`. NOTIFY avisa al secundario, que copia el cambio en segundos. Comprobar que el serial es el mismo en ambos con `dig @<servidor> ASIR2.local SOA +short`.

### 11.7 Añadir una zona nueva a un secundario que ya funciona

Caso habitual: el secundario ya está montado y se crea después una zona nueva en el primario (por ejemplo `jordi.asir2.iespoveda.local`, apartado 10). Solo hay que autorizarla en el primario y declararla en el secundario. **No se repite el `sed` del apartado 11.1**, porque duplicaría las líneas de las otras zonas.

**En el primario.** Autorizar al secundario solo en la zona nueva (la primera línea evita hacerlo dos veces):

```
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak2
grep -A4 'zone "jordi.asir2.iespoveda.local"' /etc/bind/named.conf.local | grep -q allow-transfer || sudo sed -i '/zone "jordi.asir2.iespoveda.local"/,/};/ s|type primary;|type primary;\n    allow-transfer { 172.16.6.30; };\n    also-notify { 172.16.6.30; };|' /etc/bind/named.conf.local
grep -c 'allow-transfer' /etc/bind/named.conf.local
sudo named-checkconf && sudo rndc reload
```

✅ El recuento debe dar una línea por zona (`3` con directa, inversa y `jordi.asir2.iespoveda.local`).

**En el secundario.** Declarar la zona nueva (la primera línea evita duplicarla) y pedir la transferencia:

```
grep -q 'zone "jordi.asir2.iespoveda.local"' /etc/bind/named.conf.local || sudo tee -a /etc/bind/named.conf.local > /dev/null << 'EOF'

zone "jordi.asir2.iespoveda.local" {
    type secondary;
    primaries { 172.16.6.20; };
    file "/var/cache/bind/db.jordi.asir2.iespoveda.local";
};
EOF

sudo named-checkconf && sudo rndc reconfig
sleep 3
sudo rndc retransfer jordi.asir2.iespoveda.local
```

Comprobar:

```
journalctl -u named --since "2 min ago" --no-pager | grep -i -E "jordi.asir2.iespoveda.local.*(transfer|loaded)"
dig @172.16.6.30 jordi.asir2.iespoveda.local SOA +short
dig @172.16.6.20 jordi.asir2.iespoveda.local SOA +short
dig @172.16.6.30 www.jordi.asir2.iespoveda.local +short
```

✅ En el log, `Transfer status: success`; los dos `SOA` con el **mismo serial**; y la última consulta devuelve `172.16.6.20`.


## Operaciones habituales

```
equipo9 IN A   172.16.6.29                 ; en db.ASIR2.local
29      IN PTR equipo9.ASIR2.local.        ; en db.172.16.6
```

Añadir un equipo: un `A` en la zona directa y un `PTR` en la inversa. Después, **subir el serial en ambos ficheros**, validar con `named-checkzone` y recargar.

```
sudo rndc reload
```

Recarga zonas y configuración sin cortar el servicio.

```
sudo rndc status
```

Estado del servidor.

```
sudo rndc flush
```

Vacía la caché de respuestas externas.

```
sudo journalctl -u named -f
```

Log en tiempo real.

```
dig @172.16.6.20 ASIR2.local SOA
dig @172.16.6.20 ASIR2.local NS
```

Consultar un tipo de registro concreto (útil para comprobar el serial cargado).

```
www IN CNAME equipo1
```

Alias: `www.ASIR2.local` apunta a lo mismo que `equipo1`.

Cambiar los forwarders: editar `forwarders` en `named.conf.options`, ejecutar `sudo named-checkconf` y `sudo systemctl reload named`.

### Cambiar la IP o la red del servidor

Cambiar la IP en netplan **no basta**: hay que actualizar también BIND y el DHCP. Ejemplo de `172.16.5.x` a `172.16.6.x`:

```
sudo sed -i 's/172\.16\.5/172.16.6/g' /etc/bind/named.conf.options
sudo sed -i -e 's/5\.16\.172\.in-addr\.arpa/6.16.172.in-addr.arpa/' -e 's/db\.172\.16\.5/db.172.16.6/' /etc/bind/named.conf.local
sudo mv /etc/bind/zones/db.172.16.5 /etc/bind/zones/db.172.16.6
sudo sed -i -e 's/172\.16\.5\./172.16.6./' -e 's/2026101301/2026101302/' /etc/bind/zones/db.ASIR2.local
sudo sed -i 's/2026101301/2026101302/' /etc/bind/zones/db.172.16.6
sudo named-checkconf
sudo named-checkzone ASIR2.local /etc/bind/zones/db.ASIR2.local
sudo named-checkzone 6.16.172.in-addr.arpa /etc/bind/zones/db.172.16.6
sudo systemctl restart named
```

Comprobar el resultado, incluida la salida a internet:

```
dig @172.16.6.20 equipo1.ASIR2.local +short
ip route | grep default
ping -c 2 8.8.8.8
```

La **puerta de enlace** debe ser la real de la nueva red (en otro equipo Windows de la misma red, `ipconfig`; en Linux, `ip route | grep default`). Si no lo es, el servidor llega a la red local pero no a internet, y el reenvío a Google falla.

Qué hace cada línea, por orden: actualiza `listen-on` y `allow-*`; renombra la zona inversa y su fichero en `named.conf.local`; renombra el fichero de la zona inversa; cambia las IPs de los registros `A` y sube el serial; sube el serial de la zona inversa (sus `PTR` no cambian, solo llevan el último octeto); valida y reinicia. Adaptar los números a la red real.

Con un secundario montado, actualizar también `allow-transfer` y `also-notify` en el primario, y `primaries` y `listen-on` en el secundario. Después, actualizar la subred, el pool y el gateway del DHCP, y las reservas.

Para encontrar la IP antigua en el resto de configuraciones de la máquina (ejecutar en cada máquina con servicios):

```
sudo grep -rn "172\.16\.5" /etc/bind /etc/kea /etc/nginx /etc/netplan
```

Solo deben quedar copias `.bak`. Se ejecuta sin `2>/dev/null`: ocultaría los errores de carpetas inexistentes.


---

| Síntoma | Causa probable |
|---|---|
| `NXDOMAIN` para un equipo | Registro ausente, falta el punto final en un FQDN, o no se subió el serial / no se recargó. |
| `SERVFAIL` en dominios externos | Forwarders inalcanzables (sin salida a internet o firewall). |
| `REFUSED` | El cliente está fuera de `allow-query` / `allow-recursion`. |
| Timeout sin respuesta | Firewall (`sudo ufw allow 53`), o la IP del servidor no está en `listen-on`. Comprobar con `ip -4 a` y `sudo ss -tulpn \| grep named`. También si el cliente está en otra red distinta a la del servidor. |
| Timeout tras cambiar la IP del servidor | `listen-on` y las zonas siguen con la IP o red antigua (ver "Cambiar la IP o la red del servidor"). |
| `dig` sin `@` no devuelve nada | El cliente usa otro DNS (por ejemplo `8.8.8.8`). Comprobar con `resolvectl status`. |
| Respuestas inconsistentes o *timeouts* intermitentes | IP duplicada: dos máquinas (por ejemplo clones) con la misma IP. Cada máquina debe tener la suya. |
| `dig` funciona pero `ping nombre.local` falla | `avahi`/`nss-mdns` intercepta los `.local`; no es un fallo del DNS. |
| `systemctl is-enabled bind9` responde `alias` | Es normal: el servicio real es `named`; comprobar con `systemctl is-enabled named`. |
| El servicio no arranca | Error en la configuración: `sudo named-checkconf` y `journalctl -u named -e`. Si no hay error visible, revisar AppArmor: `sudo dmesg \| grep -i apparmor \| grep named`. |
| La búsqueda inversa no resuelve | Nombre de la zona inversa con los octetos sin invertir, o falta el punto final en el `PTR`. |
| `named-checkzone` avisa de datos fuera de zona | Falta un punto final y el nombre se ha duplicado (`....ASIR2.local.ASIR2.local.`). |
| `connection refused` en el log del secundario | El primario no escucha en esa IP: revisar su `listen-on` con `sudo ss -tulpn \| grep named` y que `named` esté activo. No es lo mismo que `REFUSED`. |
| `Transfer status: REFUSED` | El primario rechaza la IP de origen del secundario: revisar `allow-transfer` y, si el secundario tiene varias IPs, fijar `transfer-source`. |
| El secundario no se actualiza | No se subió el serial en el primario, o el primario no puede avisarle (firewall en TCP 53). Forzar con `sudo rndc retransfer ASIR2.local`. |
| `permission denied` al guardar la zona en el secundario | El `file` de la zona apunta a `/etc/bind`; AppArmor solo deja escribir en `/var/cache/bind`. |
| El secundario deja de responder tras un tiempo | Pasó el `Expire` del `SOA` sin contactar con el primario. |
| `named` no arranca con `'allow-transfer' redefined` | Líneas duplicadas en `named.conf.local` por ejecutar dos veces el `sed` del apartado 11.1. Eliminar duplicados (ver más abajo), validar y arrancar. |
| El servidor no sale a internet tras cambiar la red | Puerta de enlace incorrecta: `ip route \| grep default` y `ping -c 2 <puerta de enlace>` |
| Un cliente sigue viendo la IP o la web antigua | Caché de DNS del cliente: en Windows `Clear-DnsClientCache`; en el navegador `Ctrl+F5`. O el cliente tiene otro DNS delante: `nslookup <nombre> 172.16.6.20` pregunta directamente al servidor |
| `curl http://nombre` no devuelve nada en el propio servidor | El servidor no usa su propio DNS: probar con `curl -sS -H "Host: <nombre>" http://<IP>` |
| Un bloque de comandos no hace efecto | Se ejecutó en la máquina equivocada: comprobar `hostname` antes de lanzar cada bloque, sobre todo entre primario y secundario. |

### Recuperar `named` tras un `allow-transfer` duplicado

Conserva una línea por zona. Sustituir la IP si el secundario es otro:

```
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.duplicado
sudo awk '/^zone/{a=0;n=0} /allow-transfer/{if(a++)next} /also-notify/{if(n++)next} {print}' /etc/bind/named.conf.local.duplicado | sudo tee /etc/bind/named.conf.local > /dev/null
sudo named-checkconf && echo "CONFIG OK"
sudo systemctl reset-failed named
sudo systemctl start named
```
