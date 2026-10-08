# Instalación y configuración de DNS con BIND9 (Ubuntu)

> Antes de instalar: comprobar que no hay otro servicio DNS escuchando en el puerto 53 de la IP del servidor.
>
> ```bash
> sudo ss -tulpn | grep ':53'
> ```
>
> En Ubuntu, `systemd-resolved` escucha solo en `127.0.0.53`; no entra en conflicto con BIND si este escucha en otras direcciones.

## Conceptos

| Concepto | Qué es |
|---|---|
| Servidor autoritativo | Responde con datos de las zonas que gestiona él mismo (respuesta con flag `aa`). |
| Servidor recursivo | Busca respuestas de dominios que no son suyos en nombre del cliente (flag `ra` = recursión disponible). |
| Forwarder (reenviador) | Servidor al que se reenvían las consultas que no son de zonas propias. Aquí, Google (`8.8.8.8`, `8.8.4.4`). |
| Zona | Parte del espacio de nombres que este servidor gestiona (`aula.internal`). |
| Zona directa | Traduce nombre → IP. Usa registros `A`. |
| Zona inversa | Traduce IP → nombre. Usa registros `PTR`. Su nombre es la red con los octetos al revés + `.in-addr.arpa` (`172.16.5.0/24` → `5.16.172.in-addr.arpa`). |
| SOA | Primer registro de toda zona: servidor primario, contacto del administrador (el primer punto equivale a `@`), serial y temporizadores. |
| Serial | Versión de la zona. Se incrementa en cada cambio (convención `AAAAMMDDNN`). |
| NS | Indica qué servidor es la autoridad de la zona. |
| A / PTR / CNAME | Nombre → IPv4 / IP → nombre / alias de otro nombre. |
| TTL | Segundos que otros pueden guardar en caché una respuesta. |
| FQDN y punto final | Un nombre completo termina en punto (`equipo1.aula.internal.`). Sin punto, BIND le añade el nombre de la zona y queda duplicado. |

Reglas de nombres:
- Los nombres de host no admiten espacios: `equipo1`, `equipo2`... (o `equipo-1`).
- Evitar el sufijo `.local` (reservado para mDNS). Se usa `aula.internal`.

## 1. Instalar

```bash
sudo apt update
sudo apt install bind9 bind9-utils bind9-dnsutils
```

`bind9-utils` aporta los comandos `named-checkconf` y `named-checkzone`; `bind9-dnsutils` aporta `dig` y `nslookup`.

## 2. Opciones generales y reenvío

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
sudo tee /etc/bind/named.conf.options > /dev/null << 'EOF'
options {
    directory "/var/cache/bind";

    // IPs por las que escucha
    listen-on { 127.0.0.1; 172.16.5.20; };
    listen-on-v6 { none; };

    // Quién puede consultar y quién puede usar recursión
    allow-query     { localhost; 172.16.5.0/24; };
    allow-recursion { localhost; 172.16.5.0/24; };

    // Lo que no sea de zonas propias se reenvía a Google
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;

    dnssec-validation auto;
};
EOF
```

| Opción | Para qué sirve |
|---|---|
| `listen-on` | IPs del servidor donde atiende peticiones. |
| `allow-query` / `allow-recursion` | Limitan quién consulta y quién puede pedir resolución externa. Sin límite, el servidor sería un *resolver abierto* (abusable desde internet). |
| `forwarders` | Servidores a los que se reenvía lo que no es de zonas propias. |
| `forward only` | Solo usa los forwarders. Con `forward first`, si fallan, intentaría resolver por su cuenta. |
| `dnssec-validation auto` | Valida firmas DNSSEC de las respuestas externas. |

## 3. Declarar las zonas

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
sudo tee -a /etc/bind/named.conf.local > /dev/null << 'EOF'

zone "aula.internal" {
    type primary;
    file "/etc/bind/zones/db.aula.internal";
};

zone "5.16.172.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.172.16.5";
};
EOF
```

`type primary` indica que este servidor guarda la zona original (no una copia de otro).

## 4. Crear los ficheros de zona

```bash
sudo mkdir -p /etc/bind/zones
```

**Zona directa** (nombre → IP):

```bash
sudo tee /etc/bind/zones/db.aula.internal > /dev/null << 'EOF'
$TTL 3600
@       IN  SOA ns1.aula.internal. admin.aula.internal. (
                2026100801  ; Serial (AAAAMMDDNN)
                3600        ; Refresh
                900         ; Retry
                604800      ; Expire
                3600 )      ; Negative cache TTL

@       IN  NS  ns1.aula.internal.

ns1     IN  A   172.16.5.20
equipo1 IN  A   172.16.5.21
equipo2 IN  A   172.16.5.22
equipo3 IN  A   172.16.5.23
equipo4 IN  A   172.16.5.24
equipo5 IN  A   172.16.5.25
equipo6 IN  A   172.16.5.26
equipo7 IN  A   172.16.5.27
equipo8 IN  A   172.16.5.28
EOF
```

**Zona inversa** (IP → nombre). Solo se escribe el último octeto; el resto lo da el nombre de la zona:

```bash
sudo tee /etc/bind/zones/db.172.16.5 > /dev/null << 'EOF'
$TTL 3600
@   IN  SOA ns1.aula.internal. admin.aula.internal. (
            2026100801  ; Serial (AAAAMMDDNN)
            3600        ; Refresh
            900         ; Retry
            604800      ; Expire
            3600 )      ; Negative cache TTL

@   IN  NS  ns1.aula.internal.

20  IN  PTR ns1.aula.internal.
21  IN  PTR equipo1.aula.internal.
22  IN  PTR equipo2.aula.internal.
23  IN  PTR equipo3.aula.internal.
24  IN  PTR equipo4.aula.internal.
25  IN  PTR equipo5.aula.internal.
26  IN  PTR equipo6.aula.internal.
27  IN  PTR equipo7.aula.internal.
28  IN  PTR equipo8.aula.internal.
EOF
```

## 5. Validar

```bash
sudo named-checkconf
sudo named-checkzone aula.internal /etc/bind/zones/db.aula.internal
sudo named-checkzone 5.16.172.in-addr.arpa /etc/bind/zones/db.172.16.5
```

`named-checkconf` no muestra nada si todo está bien. `named-checkzone` debe terminar en `OK`.

## 6. Arrancar y comprobar

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
systemctl is-enabled bind9
```

Debe salir `active (running)` y `enabled` (viene habilitado por defecto). Si falla: `journalctl -u bind9 -e`

Si el firewall `ufw` está activo:

```bash
sudo ufw allow 53
```

## 7. Pruebas

```bash
dig @172.16.5.20 equipo1.aula.internal +short
dig @172.16.5.20 -x 172.16.5.21 +short
dig @172.16.5.20 google.com +short
```

Resultados esperados:
- Primera consulta: `172.16.5.21` (zona directa).
- Segunda: `equipo1.aula.internal.` (zona inversa).
- Tercera: una o varias IPs de Google (reenvío).

Comprobar los 8 equipos de una vez:

```bash
for i in $(seq 1 8); do dig @172.16.5.20 equipo$i.aula.internal +short; done
```

Distinguir respuesta propia de reenviada:

```bash
dig @172.16.5.20 equipo1.aula.internal | grep -E 'status|flags'
dig @172.16.5.20 google.com | grep -E 'status|flags'
```

En la primera aparece `aa` (autoritativa); en la segunda no (viene del forwarder). Ambas deben dar `status: NOERROR`.

Desde un cliente Windows:

```powershell
nslookup equipo1.aula.internal 172.16.5.20
```

## 8. Configurar los clientes

**Por DHCP (Kea)**, en `option-data` de la subred:

```json
{ "name": "domain-name-servers", "data": "172.16.5.20" },
{ "name": "domain-name", "data": "aula.internal" }
```

**Por DHCP (Windows Server):**

```powershell
Set-DhcpServerv4OptionValue -ScopeId <RED> -DnsServer 172.16.5.20 -DnsDomain aula.internal
```

**Cliente Ubuntu con IP fija (netplan):**

```yaml
nameservers:
  addresses: [172.16.5.20]
  search: [aula.internal]
```

Los registros DNS son fijos, así que cada equipo debe conservar siempre su IP. Reservarla por MAC en el DHCP (Kea, dentro de `subnet4`):

```json
"reservations": [
  { "hw-address": "08:00:27:aa:bb:01", "ip-address": "172.16.5.21", "hostname": "equipo1" }
]
```

(una entrada por equipo).

## Operaciones habituales

```text
equipo9 IN A   172.16.5.29                 ; en db.aula.internal
29      IN PTR equipo9.aula.internal.      ; en db.172.16.5
```
Añadir un equipo: un `A` en la zona directa y un `PTR` en la inversa. Después, **subir el serial en ambos ficheros**, validar con `named-checkzone` y recargar.

```bash
sudo rndc reload
```
Recarga zonas y configuración sin cortar el servicio.

```bash
sudo rndc status
```
Estado del servidor.

```bash
sudo rndc flush
```
Vacía la caché de respuestas externas.

```bash
sudo journalctl -u bind9 -f
```
Log en tiempo real.

```bash
dig @172.16.5.20 aula.internal SOA
dig @172.16.5.20 aula.internal NS
```
Consultar un tipo de registro concreto (útil para comprobar el serial cargado).

```text
www IN CNAME equipo1
```
Alias: `www.aula.internal` apunta a lo mismo que `equipo1`.

Cambiar los forwarders: editar `forwarders` en `named.conf.options`, ejecutar `sudo named-checkconf` y `sudo systemctl reload bind9`.

---

| Síntoma | Causa probable |
|---|---|
| `NXDOMAIN` para un equipo | Registro ausente, falta el punto final en un FQDN, o no se subió el serial / no se recargó. |
| `SERVFAIL` en dominios externos | Forwarders inalcanzables (sin salida a internet o firewall). |
| `REFUSED` | El cliente está fuera de `allow-query` / `allow-recursion`. |
| Timeout sin respuesta | Firewall (`sudo ufw allow 53`) o la IP del servidor no está en `listen-on`. |
| El servicio no arranca | Error en la configuración: `sudo named-checkconf` y `journalctl -u bind9 -e`. Si no hay error visible, revisar AppArmor: `sudo dmesg \| grep -i apparmor \| grep named`. |
| La búsqueda inversa no resuelve | Nombre de la zona inversa con los octetos sin invertir, o falta el punto final en el `PTR`. |
| `named-checkzone` avisa de datos fuera de zona | Falta un punto final y el nombre se ha duplicado (`....aula.internal.aula.internal.`). |
