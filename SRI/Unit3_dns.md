# Instalación y configuración de DNS con BIND9 (Ubuntu)

> Antes de instalar: comprobar que no hay otro servicio DNS escuchando en el puerto 53 de la IP del servidor.
>
> ```
> sudo ss -tulpn | grep ':53'
> ```
>
> En Ubuntu, `systemd-resolved` escucha solo en `127.0.0.53`; no entra en conflicto con BIND si este escucha en otras direcciones.

## Cómo se forman los nombres de dominio

Un nombre de dominio se lee **de derecha a izquierda**, de lo más general a lo más concreto. Cada tramo separado por puntos es una *etiqueta*.

```
equipo1 . ASIR2 . local .
host      dominio  TLD   raíz
```

| Nivel | Qué es | Ejemplo |
|---|---|---|
| Raíz | Origen de todo el árbol; es el punto final del nombre | `.` |
| Dominio de primer nivel (TLD) | Genérico, de país o especial | `com`, `org`, `es`, `local` |
| Dominio de segundo nivel | El que registra una organización | `google.com`, `ASIR2.local` |
| Subdominio / host | Lo que cada organización crea dentro de su dominio | `www.google.com`, `equipo1.ASIR2.local` |

**A nivel global (DNS público):** hay un único árbol jerárquico y su gestión está **delegada**. La raíz la coordinan IANA/ICANN, cada TLD lo gestiona un registro, y las organizaciones registran sus dominios de segundo nivel a través de un registrador. Cada nivel indica quién es el siguiente mediante registros `NS`: para resolver `www.google.com`, un servidor pregunta a la raíz, esta lo remite al servidor de `com`, y este al de `google.com`.

**A nivel local (red privada):** la organización monta su propio servidor DNS y define una zona propia que solo se resuelve dentro de su red. No hace falta registrarla, pero no debe coincidir con un dominio público que se quiera seguir usando. Sufijos pensados para esto:

| Sufijo | Uso |
|---|---|
| `.internal` | Reservado para uso privado. |
| `.home.arpa` | Reservado para redes domésticas (RFC 8375). |
| `.local` | Reservado para mDNS (RFC 6762). En esta práctica se usa `ASIR2.local.` por indicación del docente. |

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

Qué hace cada línea, por orden: actualiza `listen-on` y `allow-*`; renombra la zona inversa y su fichero en `named.conf.local`; renombra el fichero de la zona inversa; cambia las IPs de los registros `A` y sube el serial; sube el serial de la zona inversa (sus `PTR` no cambian, solo llevan el último octeto); valida y reinicia. Adaptar los números a la red real. Después, actualizar también la subred, el pool y el gateway del DHCP, y las reservas.

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
