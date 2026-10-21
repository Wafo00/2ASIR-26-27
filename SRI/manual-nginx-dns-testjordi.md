# Nginx con dos virtual hosts y DNS propio (BIND9) en Ubuntu Server

**Objetivo:** servir dos webs desde un mismo servidor Nginx y acceder a ellas **por nombre, sin escribir IPs**, resolviendo los nombres con un servidor DNS (BIND9) instalado en la propia máquina.

| Sitio | Dirección | Puerto |
|---|---|---|
| 1 | `http://<DOMINIO1>` | 80 |
| 2 | `http://<DOMINIO2>:<PUERTO2>` | el segundo puerto |

Los clientes que quieran entrar deben usar la IP del servidor como **servidor DNS** (ver [apartado 10](#10-acceso-desde-otro-equipo)).

---

## 0. Definir las variables (una sola vez)

Cambiad los valores de ejemplo por los vuestros y pegad el bloque en la terminal. **El resto del manual se copia tal cual, sin editar nada.**

```bash
IP=192.168.1.10
D1=web1.test
D2=web2.test
S1=sitio1
S2=sitio2
P2=8080
```

| Variable | Qué es | Ejemplo |
|---|---|---|
| `IP` | IP **fija** del servidor | `192.168.1.10` |
| `D1`, `D2` | Nombres de los dos sitios (usad el sufijo `.test`, reservado para pruebas) | `web1.test` |
| `S1`, `S2` | Nombre de carpeta y de fichero de cada sitio | `sitio1` |
| `P2` | Puerto del segundo sitio | `8080` |

Comprobad que se han guardado:
```bash
echo $IP $D1 $D2 $S1 $S2 $P2
```

En los textos y comandos pensados para **otro equipo** (donde no existen estas variables), `<IP_SERVIDOR>` equivale a `IP`, `<DOMINIO1>` a `D1`, `<DOMINIO2>` a `D2` y `<PUERTO2>` a `P2`.

> ⚠️ Las variables **solo viven en esta terminal**. Si abrís otra o reiniciáis, repetid este bloque antes de seguir. Con variables vacías, los ficheros se escriben con huecos y Nginx/BIND dan error.
>
> ⚠️ **No peguéis notas ni comentarios dentro de los bloques `<<EOF`.** Todo lo que haya entre `EOF` y `EOF` acaba escrito en el fichero.

---

## 1. Instalar Nginx

```bash
sudo apt update && sudo apt install -y nginx
```

## 2. Crear las dos webs

```bash
sudo mkdir -p /var/www/${S1} /var/www/${S2}

sudo tee /var/www/${S1}/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>${S1} - puerto 80</title></head>
<body><h1>Sitio ${S1}: puerto 80</h1></body>
</html>
EOF

sudo tee /var/www/${S2}/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>${S2} - puerto ${P2}</title></head>
<body><h1>Sitio ${S2}: puerto ${P2}</h1></body>
</html>
EOF
```

Cada web se escribe por separado (sin `cp` + `sed`) para que una no herede textos de la otra.

## 3. Crear los virtual hosts

El `server_name` es el **nombre** del sitio, no la IP.

```bash
sudo tee /etc/nginx/sites-available/${S1} > /dev/null <<EOF
server {
    listen 80;
    server_name ${D1};
    root /var/www/${S1};
    index index.html;
}
EOF

sudo tee /etc/nginx/sites-available/${S2} > /dev/null <<EOF
server {
    listen ${P2};
    server_name ${D2};
    root /var/www/${S2};
    index index.html;
}
EOF
```

## 4. Activarlos y quitar el sitio por defecto

```bash
sudo ln -s /etc/nginx/sites-available/${S1} /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/${S2} /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` debe decir **syntax is ok** y **test is successful**. Si no, no sigáis: el error indica fichero y línea.

## 5. Instalar BIND9

```bash
sudo apt install -y bind9 bind9-utils dnsutils
sudo systemctl status named --no-pager
```

Debe aparecer `active (running)`. En Ubuntu el servicio se llama `named`; `bind9` es solo un alias.

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

- `forwarders` + `forward only`: lo que no sea nuestro dominio se pregunta a un DNS público.
- `dnssec-validation no`: en algunas redes las consultas a los servidores raíz fallan (`FORMERR`) y, sin ellos, BIND rechaza hasta las respuestas correctas (`SERVFAIL`). El DNS público al que se reenvía ya valida DNSSEC por su cuenta.

## 7. Crear las zonas

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
sudo mkdir -p /etc/bind/zones

sudo tee /etc/bind/named.conf.local > /dev/null <<EOF
zone "${D1}" {
        type master;
        file "/etc/bind/zones/db.${D1}";
};

zone "${D2}" {
        type master;
        file "/etc/bind/zones/db.${D2}";
};
EOF

sudo tee /etc/bind/zones/db.${D1} > /dev/null <<EOF
\$TTL 604800
@   IN  SOA ns.${D1}. admin.${D1}. (
        1 604800 86400 2419200 604800 )
@   IN  NS  ns.${D1}.
ns  IN  A   ${IP}
@   IN  A   ${IP}
EOF

sudo tee /etc/bind/zones/db.${D2} > /dev/null <<EOF
\$TTL 604800
@   IN  SOA ns.${D2}. admin.${D2}. (
        1 604800 86400 2419200 604800 )
@   IN  NS  ns.${D2}.
ns  IN  A   ${IP}
@   IN  A   ${IP}
EOF
```

Cada zona dice: "este dominio es mío (`ns`) y el nombre a secas apunta a la IP del servidor". La barra de `\$TTL` es necesaria: evita que la terminal interprete `$TTL` como una variable.

Comprobar y aplicar:

```bash
sudo named-checkconf
sudo named-checkzone ${D1} /etc/bind/zones/db.${D1}
sudo named-checkzone ${D2} /etc/bind/zones/db.${D2}
sudo systemctl restart named
```

`named-checkconf` no debe mostrar nada y los dos `named-checkzone` deben terminar en `OK`.

## 8. Comprobar en el servidor

**DNS** (preguntando por su IP):
```bash
dig @${IP} ${D1} +short
dig @${IP} ${D2} +short
dig @127.0.0.1 www.google.com | grep -E "status|^www"
```
Los dos primeros devuelven la IP del servidor. El tercero, `status: NOERROR` y las IPs.

**Webs** (indicando el nombre a mano):
```bash
curl -H "Host: ${D1}" http://${IP}
curl -H "Host: ${D2}" http://${IP}:${P2}
```
Deben salir la web 1 y la web 2.

**Servicios y firewall:**
```bash
sudo systemctl is-enabled nginx named
sudo ufw status
```
`enabled` dos veces. Si UFW está `inactivo` no hay que hacer nada; si estuviera activo:
```bash
sudo ufw allow 53
sudo ufw allow 80/tcp
sudo ufw allow ${P2}/tcp
```

## 9. IP fija

Los nombres apuntan a una IP: si esta cambia, los clientes dejan de entrar. La IP del servidor debe ser **estática**.

Después de un reinicio, comprobad que sigue ahí:
```bash
ip -4 -br a
```
Debe mostrarse `<IP>/<máscara>`. Si la interfaz también recibe una segunda dirección por DHCP, que cambia entre arranques, no afecta mientras los nombres apunten a la IP fija.

## 10. Acceso desde otro equipo

1. El equipo debe poder **llegar a la IP del servidor** (misma red o con ruta entre ambas).
2. Poner la IP del servidor como **servidor DNS** del equipo (adaptador de red → IPv4 → DNS preferido).
3. Abrir en el navegador, **con `http://` delante**:
   - `http://<DOMINIO1>`
   - `http://<DOMINIO2>:<PUERTO2>`

Sin cambiar nada, se puede probar solo la resolución indicando el servidor al final:
```text
nslookup <DOMINIO1> <IP_SERVIDOR>
nslookup <DOMINIO2> <IP_SERVIDOR>
```
Cada uno debe responder con la IP del servidor. (Un aviso inicial de "No se encuentra el nombre del servidor" es normal: no hay zona inversa.)

## 11. Problemas típicos

| Síntoma | Causa típica |
|---|---|
| `nginx -t`: `unknown directive "(ip"` u otra directiva extraña | Texto suelto pegado dentro de un bloque `server` |
| Ficheros con huecos (`server_name ;`, rutas vacías) | Variables sin definir: repetir el apartado 0 |
| Sale "Welcome to nginx" | No se borró el `default` (apartado 4) |
| Las dos webs muestran lo mismo | `root` mal escrito, o `index.html` repetido en las dos carpetas |
| `dig` a internet da `SERVFAIL` | Validación DNSSEC activa: usar `dnssec-validation no;` y `forward only;` (apartado 6) |
| `is-enabled bind9` responde `alias` | Normal: el servicio real es `named` |
| El nombre no resuelve en el cliente | El cliente no usa la IP del servidor como DNS |
| El navegador busca en Google en vez de abrir la web | Falta `http://` delante |
| El segundo sitio no carga | Falta `:<PUERTO2>` en la dirección |
| `nslookup` sin servidor indicado da error en Windows | Usa el DNS del adaptador, no el nuestro: indicar la IP del servidor al final |

## 12. Deshacer (opcional)

```bash
sudo cp /etc/bind/named.conf.options.bak /etc/bind/named.conf.options
sudo cp /etc/bind/named.conf.local.bak /etc/bind/named.conf.local
sudo rm -f /etc/bind/zones/db.${D1} /etc/bind/zones/db.${D2}
sudo systemctl restart named
```
Para quitar BIND por completo: `sudo apt purge -y bind9 bind9-utils`.
