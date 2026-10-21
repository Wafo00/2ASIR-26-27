# Servidor DNS con BIND9 en Ubuntu Server (con web de prueba en Nginx)

**Objetivo:** montar desde cero un servidor DNS propio que resuelva los nombres de un dominio local (`<host>.<ZONA>`) y gestionarlo en el día a día: añadir, consultar, cambiar y eliminar dominios. Nginx se usa solo como **web de prueba** para comprobar que los nombres llevan a donde deben.

**Diseño:** una **única zona** (`<ZONA>`, p. ej. `grupo.test`) con un registro por dominio. Dar de alta una web = **una línea** en un fichero.

```text
Cliente --(consulta DNS)--> Servidor BIND9 --> "web1.grupo.test = 192.168.1.10"
Cliente --(HTTP a esa IP)--> Servidor web (este u otro equipo)
```

**Requisitos:** Ubuntu Server con `sudo` y acceso a internet.

**Índice:** 0 Variables · 1 IP fija · 2 Instalar BIND9 · 3 Opciones · 4 Zona · 5 Comprobar · 6 Cortafuegos · 7 Clientes · **8 Operaciones habituales** · 9 Web de prueba · 10 Problemas · 11 Deshacer · Anexo ficheros

---

## 0. Definir las variables (una sola vez)

Cambiad los valores de ejemplo por los vuestros y pegad el bloque en la terminal. **El resto del manual se copia tal cual, sin editar nada.**

```bash
IP=192.168.1.10
MASCARA=24
GW=192.168.1.1
IFACE=enp0s3
ZONA=grupo.test
H1=web1
H2=web2
```

| Variable | Qué es | Cómo averiguarlo |
|---|---|---|
| `IP` | IP **fija** que tendrá el servidor | La eliges tú, libre y de la red |
| `MASCARA` | Prefijo de la red | `24` equivale a 255.255.255.0 |
| `GW` | Puerta de enlace (router) | `ip route \| grep default` |
| `IFACE` | Nombre de la tarjeta de red | `ip -br a` |
| `ZONA` | Dominio que gestiona este DNS (sufijo `.test`, reservado para pruebas; evitad `.local`) | La eliges tú |
| `H1`, `H2` | Nombres de las dos primeras webs (quedan `web1.grupo.test`...) | Los eliges tú |

Comprobad que se han guardado:
```bash
echo $IP $MASCARA $GW $IFACE $ZONA $H1 $H2
```

En los comandos pensados para **otro equipo** (donde no existen estas variables), `<IP_SERVIDOR>` equivale a `IP`, y `<HOST>.<ZONA>` a un nombre como `${H1}.${ZONA}`.

> ⚠️ Las variables **solo viven en esta terminal**. Si abrís otra o reiniciáis, repetid este bloque antes de seguir. Con variables vacías, los ficheros se escriben con huecos y BIND da error.
>
> ⚠️ **No peguéis notas ni comentarios dentro de los bloques `<<EOF`.** Todo lo que haya entre `EOF` y `EOF` acaba escrito en el fichero.

---

## 1. IP fija

Los nombres apuntan a una IP: si cambia, los clientes dejan de entrar. Y los clientes usarán esta IP como DNS, así que **debe ser estática**.

Si ya la tiene, comprobadlo y pasad al apartado 2:
```bash
ip -4 -br a
```

Si no, configurarla con **netplan** (Ubuntu Server):
```bash
ls /etc/netplan/
```
Se verá un `.yaml` (p. ej. `50-cloud-init.yaml`). Hacer copia y sustituirlo:
```bash
F=$(ls /etc/netplan/*.yaml | head -n1)
sudo cp $F $F.bak

sudo tee $F > /dev/null <<EOF
network:
  version: 2
  ethernets:
    ${IFACE}:
      dhcp4: false
      addresses: [${IP}/${MASCARA}]
      routes:
        - to: default
          via: ${GW}
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
EOF

sudo chmod 600 $F
sudo netplan try
```
`netplan try` aplica el cambio y, si no se confirma con Enter en 120 s, **lo revierte solo** (útil si se trabaja por SSH).

> Si el sistema usa **NetworkManager** (típico de Ubuntu Desktop) el equivalente es:
> `sudo nmcli con mod "<nombre-conexión>" ipv4.method manual ipv4.addresses ${IP}/${MASCARA} ipv4.gateway ${GW} ipv4.dns "8.8.8.8"` y después `sudo nmcli con up "<nombre-conexión>"`. Los nombres de conexión salen de `nmcli con show`.

Comprobar, también tras reiniciar:
```bash
ip -4 -br a
ping -c 2 8.8.8.8
```
Debe verse `<IP>/<máscara>` y haber respuesta. Si la interfaz recibe además una segunda dirección por DHCP, no afecta mientras los nombres apunten a la IP fija.

## 2. Instalar BIND9

```bash
sudo apt update && sudo apt install -y bind9 bind9-utils dnsutils
sudo systemctl status named --no-pager
```

Debe aparecer `active (running)`. En Ubuntu el servicio se llama `named`; `bind9` es solo un alias. Los ficheros están en `/etc/bind/` (ver el [anexo](#anexo-ficheros-de-configuración-de-bind9)).

## 3. Configurar BIND9 (opciones)

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

- `forwarders` + `forward only`: lo que no sea nuestro dominio se pregunta a un DNS público. Así el servidor sigue resolviendo internet.
- `dnssec-validation no`: en algunas redes las consultas a los servidores raíz fallan (`FORMERR`) y, sin ellos, BIND rechaza hasta las respuestas correctas (`SERVFAIL`). El DNS público al que se reenvía ya valida DNSSEC por su cuenta.

## 4. Crear la zona

Se declara **una zona** y se crea su fichero, con un registro `A` por dominio.

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
sudo mkdir -p /etc/bind/zones

sudo tee /etc/bind/named.conf.local > /dev/null <<EOF
// ===== ${ZONA} =====
zone "${ZONA}" {
        type master;
        file "/etc/bind/zones/db.${ZONA}";
};
EOF

sudo tee /etc/bind/zones/db.${ZONA} > /dev/null <<EOF
\$TTL 604800
@     IN  SOA ns.${ZONA}. admin.${ZONA}. (
          1 604800 86400 2419200 604800 )
@     IN  NS  ns.${ZONA}.
ns    IN  A   ${IP}
${H1} IN  A   ${IP}
${H2} IN  A   ${IP}
EOF
```

- La zona dice: "este dominio es mío (`ns`) y cada nombre de la lista apunta a su IP".
- La barra de `\$TTL` es necesaria: evita que la terminal interprete `$TTL` como una variable.
- Cada línea `nombre IN A ip` es un dominio. **Ese es el fichero que se toca en el día a día** (apartado 8).

Comprobar y aplicar:
```bash
sudo named-checkconf
sudo named-checkzone ${ZONA} /etc/bind/zones/db.${ZONA}
sudo systemctl restart named
```
`named-checkconf` no debe mostrar nada y `named-checkzone` debe terminar en `OK`.

## 5. Comprobar en el servidor

```bash
dig @${IP} ${H1}.${ZONA} +short
dig @${IP} ${H2}.${ZONA} +short
dig @127.0.0.1 www.google.com | grep -E "status|^www"
sudo systemctl is-enabled named
```
- Los dos primeros devuelven la IP del servidor.
- El tercero, `status: NOERROR` y las IPs (comprueba el reenvío a internet).
- `enabled`: arrancará solo tras reiniciar.

## 6. Cortafuegos

```bash
sudo ufw status
```
Si está `inactivo` no hay que hacer nada. Si estuviera activo, abrir el DNS (y la web de prueba si se usa):
```bash
sudo ufw allow 53
sudo ufw allow 80/tcp
```

## 7. Acceso desde otro equipo

1. El equipo debe poder **llegar a la IP del servidor** (misma red o con ruta entre ambas).
2. Poner la IP del servidor como **servidor DNS** del equipo (adaptador de red → IPv4 → DNS preferido). Alternativas: que lo reparta el DHCP de la red, o que lo delegue otro DNS.
3. En el navegador, **con `http://` delante**: `http://<HOST>.<ZONA>`

Sin cambiar nada en el cliente, se prueba solo la resolución indicando el servidor al final:
```text
nslookup <HOST>.<ZONA> <IP_SERVIDOR>
```
Debe responder con la IP del dominio. (Un aviso inicial de "No se encuentra el nombre del servidor" es normal: no hay zona inversa.)

---

## 8. Operaciones habituales

Todas se hacen sobre el fichero de zona y terminan igual: **comprobar y aplicar**.

```bash
sudo named-checkzone ${ZONA} /etc/bind/zones/db.${ZONA}
sudo systemctl restart named
```

> Las variables `IP` y `ZONA` del apartado 0 deben estar definidas. Si no, repetirlas.

### 8.1 Añadir un dominio

Definir el nombre y la IP de la máquina que **aloja la web** (si es este servidor, `IP3=${IP}`):
```bash
H3=nuevo
IP3=192.168.1.20
```

Añadir el registro (solo si no existe ya):
```bash
grep -E "^${H3}[[:space:]]" /etc/bind/zones/db.${ZONA} || \
  printf '%-10s IN  A   %s\n' "${H3}" "${IP3}" | sudo tee -a /etc/bind/zones/db.${ZONA}
```

Aplicar (bloque de arriba) y verificar:
```bash
dig @${IP} ${H3}.${ZONA} +short
curl -H "Host: ${H3}.${ZONA}" http://${IP3}
```

- `dig` debe devolver `IP3`.
- `curl` debe devolver la web de esa máquina. Si está en otro puerto, añadid `:<puerto>`. El DNS **no** conoce puertos: van siempre en la URL.
- Si `dig` va bien pero `curl` falla, el problema está en el servidor web de esa máquina (parado, `nginx -t` con error, `server_name` distinto), no en este DNS.

### 8.2 Consultar los dominios

Listado del fichero (sin comentarios ni líneas vacías):
```bash
grep -vE '^\s*(;|$)' /etc/bind/zones/db.${ZONA}
```

Solo nombre completo e IP, alineado:
```bash
awk -v z="${ZONA}" '$2=="IN" && $3=="A" && $1!="ns" {print $1"."z, $4}' /etc/bind/zones/db.${ZONA} | column -t
```

Un dominio concreto, tal como lo responde el DNS (no el fichero):
```bash
dig @${IP} ${H3}.${ZONA} +short
```

Zonas cargadas en BIND:
```bash
grep -E '^zone|^//' /etc/bind/named.conf.local
sudo rndc zonestatus ${ZONA}
```

### 8.3 Cambiar la IP de un dominio

Borrar su línea y volver a añadirla con la IP nueva:
```bash
H3=nuevo
IP3=192.168.1.30
sudo sed -i "/^${H3}[[:space:]]/d" /etc/bind/zones/db.${ZONA}
printf '%-10s IN  A   %s\n' "${H3}" "${IP3}" | sudo tee -a /etc/bind/zones/db.${ZONA}
```
Aplicar y verificar con `dig`.

### 8.4 Eliminar un dominio

```bash
H3=nuevo
sudo sed -i "/^${H3}[[:space:]]/d" /etc/bind/zones/db.${ZONA}
```
Aplicar y comprobar que ya no resuelve (`dig` no devuelve nada).

### 8.5 Renombrar un dominio

Eliminar el nombre antiguo (8.4) y añadir el nuevo (8.1). Si la web es de un servidor Nginx, cambiar también su `server_name` por el nombre nuevo y recargar:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 8.6 Copia de seguridad y vuelta atrás

Antes de cambios grandes:
```bash
sudo cp /etc/bind/zones/db.${ZONA} /etc/bind/zones/db.${ZONA}.bak
```
Para volver: copiarlo de nuevo en su sitio y reiniciar `named`.

### 8.7 Varias zonas (grupos con dominio raíz distinto)

Si hace falta separar grupos (`a.test`, `b.test`), se declara una zona por grupo en `named.conf.local`, con comentario de cabecera, y un fichero `db.<zona>` por cada una:

```text
// ===== GRUPO A =====
zone "a.test" {
        type master;
        file "/etc/bind/zones/db.a.test";
};

// ===== GRUPO B =====
zone "b.test" {
        type master;
        file "/etc/bind/zones/db.b.test";
};
```

Para añadir una zona: copia de seguridad, `sudo nano /etc/bind/named.conf.local`, pegar el bloque bajo su cabecera (mejor que `tee -a`: ejecutado dos veces duplica la zona), y `sudo named-checkconf` antes de reiniciar.

---

## 9. Web de prueba con Nginx (opcional)

Sirve para comprobar que **cada nombre llega al servidor**. Una sola web de prueba responde a todos los nombres e indica con cuál se la ha llamado.

```bash
sudo apt install -y nginx

sudo tee /etc/nginx/sites-available/prueba > /dev/null <<'EOF'
server {
    listen 80 default_server;
    default_type text/plain;
    return 200 "Web de prueba. Nombre solicitado: $host\n";
}
EOF

sudo ln -sf /etc/nginx/sites-available/prueba /etc/nginx/sites-enabled/prueba
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` debe decir **syntax is ok** y **test is successful**. Comprobar:
```bash
curl -H "Host: ${H1}.${ZONA}" http://127.0.0.1
curl -H "Host: ${H2}.${ZONA}" http://127.0.0.1
```
Cada uno responde con su propio nombre. (Si no hay `curl`: `sudo apt install -y curl`.) Desde otro equipo: `http://<HOST>.<ZONA>`, con `http://` delante.

Para una web real con su propio sitio, se crea un fichero en `sites-available` con `server_name <HOST>.<ZONA>;`, su `root`, y se enlaza en `sites-enabled` como arriba.

---

## 10. Problemas típicos

| Síntoma | Causa típica |
|---|---|
| Ficheros con huecos (`ns.. `, rutas vacías) | Variables sin definir: repetir el apartado 0 |
| `dig` a internet da `SERVFAIL` | Validación DNSSEC activa: usar `dnssec-validation no;` y `forward only;` (apartado 3) |
| `is-enabled bind9` responde `alias` | Normal: el servicio real es `named` |
| El nombre no resuelve en el cliente | El cliente no usa la IP del servidor como DNS |
| `dig @IP nombre` no devuelve nada | Falta la línea del dominio en el fichero de zona, o no se reinició `named` |
| Dominio nuevo no resuelve | Se editó `named.conf.options` en vez del fichero de zona, o falta reiniciar `named` |
| `named-checkzone` da error al añadir un dominio | Línea mal escrita o nombre duplicado en el fichero de zona |
| `named-checkconf` falla tras añadir una zona | Zona declarada dos veces en `named.conf.local` |
| `dig ... AXFR` da `Transfer failed` | Normal: BIND restringe las transferencias de zona. Usar el listado de 8.2 |
| El navegador busca en Google en vez de abrir la web | Falta `http://` delante |
| El nombre resuelve, pero la web no carga | Fallo en el servidor web de esa IP, no en el DNS (8.1) |
| Sale "Welcome to nginx" en la prueba | No se borró el sitio `default` (apartado 9) |
| `nginx -t` da una directiva desconocida | Texto suelto pegado dentro del fichero del sitio |
| `nslookup` sin servidor indicado da error en Windows | Usa el DNS del adaptador, no el nuestro: indicar la IP del servidor al final |
| Tras un reinicio no funciona | IP sin fijar (apartado 1) o variables sin redefinir en la terminal |

## 11. Deshacer (opcional)

```bash
sudo cp /etc/bind/named.conf.options.bak /etc/bind/named.conf.options
sudo cp /etc/bind/named.conf.local.bak /etc/bind/named.conf.local
sudo rm -f /etc/bind/zones/db.${ZONA}
sudo systemctl restart named
```
Para quitar BIND por completo: `sudo apt purge -y bind9 bind9-utils`.
Para quitar la web de prueba: `sudo rm -f /etc/nginx/sites-enabled/prueba /etc/nginx/sites-available/prueba && sudo systemctl reload nginx`.

---

## Anexo. Ficheros de configuración de BIND9

Todos están en `/etc/bind/`:

| Para... | Fichero |
|---|---|
| Opciones generales (reenvío, DNSSEC, interfaces) | `named.conf.options` |
| **Declarar zonas** (dominios propios) | `named.conf.local` |
| Registros de cada zona (**el día a día**) | `zones/db.<zona>` (la carpeta `zones` la crea este manual) |
| Fichero principal: solo incluye a los demás, no se toca | `named.conf` |

Reglas rápidas:
- **Dar de alta una web** = una línea en `zones/db.<zona>`.
- **Dar de alta una zona nueva** = `named.conf.local` + su fichero de zona.
- **Cambiar el comportamiento del servidor** = `named.conf.options`.

Para comprobar la sintaxis de todos a la vez:
```bash
sudo named-checkconf
```
Si no imprime nada, la configuración es correcta. Tras cualquier cambio, aplicar con `sudo systemctl restart named`.
