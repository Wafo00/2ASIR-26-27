# Nginx con dos virtual hosts y DNS propio (BIND9) en Ubuntu 26.04

**Objetivo:** entrar a dos webs **por nombre, sin escribir IPs**, resolviendo los nombres con un servidor DNS montado en la propia MV.

| Sitio | Dirección | Puerto | Carpeta | Fichero de Nginx |
|---|---|---|---|---|
| A | `http://testjordi1.test` | 80 | `/var/www/dominioa` | `dominioa` |
| B | `http://testjordi2.test:8080` | 8080 | `/var/www/dominiob` | `dominiob` |

- **IP del servidor (fija):** `172.16.6.21`
- **Servidor DNS:** la misma MV (`172.16.6.21`), con BIND9.
- Los clientes que quieran entrar deben usar `172.16.6.21` como servidor DNS (ver [apartado 9](#9-cómo-acceder-desde-otro-equipo)).

> ⚠️ **No peguéis notas ni comentarios dentro de los bloques `<<'EOF'`.** Todo lo que haya entre `EOF` y `EOF` acaba escrito en el fichero, y Nginx/BIND leen hasta el último carácter.

---

## 1. Instalar Nginx

```bash
sudo apt update && sudo apt install -y nginx
```

## 2. Crear las dos webs

```bash
sudo mkdir -p /var/www/dominioa /var/www/dominiob

sudo tee /var/www/dominioa/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>Dominio A - Puerto 80</title></head>
<body><h1>Bienvenid@ a Dominio A: Puerto 80</h1></body>
</html>
EOF

sudo tee /var/www/dominiob/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>Dominio B - Puerto 8080</title></head>
<body><h1>Bienvenid@ a Dominio B: Puerto 8080</h1></body>
</html>
EOF
```

Cada web se escribe por separado (sin `cp` + `sed`) para evitar que una herede textos de la otra.

## 3. Crear los virtual hosts

El `server_name` es el **nombre** de cada sitio, no la IP.

```bash
sudo tee /etc/nginx/sites-available/dominioa > /dev/null <<'EOF'
server {
    listen 80;
    server_name testjordi1.test;
    root /var/www/dominioa;
    index index.html;
}
EOF

sudo tee /etc/nginx/sites-available/dominiob > /dev/null <<'EOF'
server {
    listen 8080;
    server_name testjordi2.test;
    root /var/www/dominiob;
    index index.html;
}
EOF
```

## 4. Activarlos y quitar el sitio por defecto

```bash
sudo ln -s /etc/nginx/sites-available/dominioa /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/dominiob /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` debe decir **syntax is ok** y **test is successful**. Si no, no sigáis: el error indica fichero y línea.

## 5. Instalar BIND9

```bash
sudo apt install -y bind9 bind9-utils dnsutils
sudo systemctl status bind9 --no-pager
```

Debe aparecer `active (running)`.

## 6. Configurar BIND9 (opciones)

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak

sudo tee /etc/bind/named.conf.options > /dev/null <<'EOF'
options {
        directory "/var/cache/bind";

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
        forward only;

        dnssec-validation no;
        listen-on-v6 { any; };
};
EOF
```

- `forwarders` + `forward only`: lo que no sea nuestro dominio se pregunta a Google.
- `dnssec-validation no`: en esta red las consultas a los servidores raíz fallan (`FORMERR`) y, sin ellos, BIND rechaza hasta las respuestas correctas (`SERVFAIL`). Google ya valida DNSSEC por su cuenta.

## 7. Crear las zonas

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak

sudo tee /etc/bind/named.conf.local > /dev/null <<'EOF'
zone "testjordi1.test" {
        type master;
        file "/etc/bind/zones/db.testjordi1.test";
};

zone "testjordi2.test" {
        type master;
        file "/etc/bind/zones/db.testjordi2.test";
};
EOF

sudo mkdir -p /etc/bind/zones

sudo tee /etc/bind/zones/db.testjordi1.test > /dev/null <<'EOF'
$TTL 604800
@   IN  SOA ns.testjordi1.test. admin.testjordi1.test. (
        1 604800 86400 2419200 604800 )
@   IN  NS  ns.testjordi1.test.
ns  IN  A   172.16.6.21
@   IN  A   172.16.6.21
EOF

sudo tee /etc/bind/zones/db.testjordi2.test > /dev/null <<'EOF'
$TTL 604800
@   IN  SOA ns.testjordi2.test. admin.testjordi2.test. (
        1 604800 86400 2419200 604800 )
@   IN  NS  ns.testjordi2.test.
ns  IN  A   172.16.6.21
@   IN  A   172.16.6.21
EOF
```

Cada zona dice: "este dominio es mío (`ns`) y el nombre a secas apunta a `172.16.6.21`".

Comprobar y aplicar:

```bash
sudo named-checkconf
sudo named-checkzone testjordi1.test /etc/bind/zones/db.testjordi1.test
sudo named-checkzone testjordi2.test /etc/bind/zones/db.testjordi2.test
sudo systemctl restart bind9
```

`named-checkconf` no debe mostrar nada y los dos `named-checkzone` deben terminar en `OK`.

## 8. Comprobar en la MV

**DNS** (por su IP):
```bash
dig @172.16.6.21 testjordi1.test +short
dig @172.16.6.21 testjordi2.test +short
dig @127.0.0.1 www.google.com | grep -E "status|^www"
```
Los dos primeros devuelven `172.16.6.21`. El tercero, `status: NOERROR` y las IPs de Google.

**Webs** (indicando el nombre a mano):
```bash
curl -H "Host: testjordi1.test" http://172.16.6.21
curl -H "Host: testjordi2.test" http://172.16.6.21:8080
```
Deben salir la web A y la web B.

**Servicios y firewall:**
```bash
sudo systemctl is-enabled nginx bind9
sudo ufw status
```
`enabled` dos veces. Si UFW está `inactivo` no hay que hacer nada; si estuviera activo:
```bash
sudo ufw allow 53
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
```

## 9. Cómo acceder desde otro equipo

1. Poner `172.16.6.21` como **servidor DNS** del equipo (adaptador de red → IPv4 → DNS preferido).
2. Abrir en el navegador, **con `http://` delante**:
   - `http://testjordi1.test`
   - `http://testjordi2.test:8080`

Sin cambiar nada, se puede probar solo la resolución:
```text
nslookup testjordi1.test 172.16.6.21
nslookup testjordi2.test 172.16.6.21
```
Cada uno debe responder `Address: 172.16.6.21`. (Un aviso inicial de "No se encuentra el nombre del servidor" es normal: no hay zona inversa.)

## 10. Problemas típicos

| Síntoma | Causa típica |
|---|---|
| `nginx -t`: `unknown directive "(ip"` | Texto suelto copiado dentro de un bloque `server` |
| Sale "Welcome to nginx" | No se borró el `default` (paso 4) |
| Las dos webs muestran lo mismo | `root` mal escrito, o `index.html` repetido en las dos carpetas |
| `dig` a internet da `SERVFAIL` | Validación DNSSEC activa: usar `dnssec-validation no;` y `forward only;` (paso 6) |
| El nombre no resuelve en el cliente | El cliente no usa `172.16.6.21` como DNS |
| El navegador busca en Google en vez de abrir la web | Falta `http://` delante |
| El segundo sitio no carga | Falta `:8080` en la dirección |
| `nslookup` sin servidor indicado da error en Windows | Usa el DNS del adaptador, no el nuestro: indicar `172.16.6.21` al final |

## 11. Deshacer (opcional)

```bash
sudo cp /etc/bind/named.conf.options.bak /etc/bind/named.conf.options
sudo cp /etc/bind/named.conf.local.bak /etc/bind/named.conf.local
sudo rm -f /etc/bind/zones/db.testjordi1.test /etc/bind/zones/db.testjordi2.test
sudo systemctl restart bind9
```
Para quitar BIND por completo: `sudo apt purge -y bind9 bind9-utils`.
