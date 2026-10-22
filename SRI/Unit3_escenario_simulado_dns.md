# Servidor DNS con BIND9 en Ubuntu Server (con web de prueba en Nginx)

**Objetivo:** montar desde cero un servidor DNS propio que resuelva los nombres de un dominio local (`<host>.<ZONA>`) y gestionarlo en el día a día: añadir, consultar, cambiar y eliminar dominios. Nginx se usa solo como **web de prueba**.

**Requisitos:** Ubuntu Server con `sudo` y acceso a internet.

```text
Cliente --(1. ¿IP de nuevo.grupo.test?)--> Servidor DNS (BIND9) --> "192.168.1.20"
Cliente --(2. abre la web en esa IP)-----> Servidor web (este equipo u otro)
```

## Cómo leer este manual

Cada apartado tiene siempre la misma forma:

> **🎯 Qué hacemos:** el objetivo del paso, en una frase.
> **📄 Archivo:** qué fichero se toca, si se **crea** o se **modifica**, y para qué sirve.

Debajo, los **comandos a copiar** y qué resultado esperar (✅).

## Mapa de archivos

Todos los ficheros que aparecen en el manual. Los de BIND están en `/etc/bind/`.

| Archivo | Para qué sirve | Se toca... |
|---|---|---|
| `/etc/netplan/*.yaml` | Fija la IP del servidor | una vez (apartado 1) |
| `named.conf.options` | Comportamiento general de BIND: reenvío a internet, DNSSEC | una vez (apartado 3) |
| `named.conf.local` | **Declara** qué zonas (dominios) gestiona este servidor | una vez, o al crear una zona nueva |
| `zones/db.<ZONA>` | **Lista de dominios y sus IP**: una línea por dominio | **a diario** (apartado 8) |
| `named.conf` | Fichero principal: solo incluye a los demás | nunca |
| `/etc/nginx/sites-available/prueba` | Web de prueba | una vez (apartado 9) |

Regla rápida: **añadir una web = una línea en `zones/db.<ZONA>`**.

---

## 0. Variables

> **🎯 Qué hacemos:** guardar en la terminal los datos propios (IP, nombre de la zona...) para no escribirlos en cada comando.
> **📄 Archivo:** ninguno. Solo se guardan en la sesión de la terminal.

Cambiad los valores de ejemplo por los vuestros y pegad el bloque. **Los apartados 1 al 7 se copian tal cual, sin editar nada.**

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

Comprobar que se han guardado:
```bash
echo $IP $MASCARA $GW $IFACE $ZONA $H1 $H2
```
✅ Deben salir los siete valores.

> ⚠️ Las variables **solo viven en esta terminal**. Si abrís otra o reiniciáis, repetid este bloque. Con variables vacías, los ficheros se escriben con huecos y BIND da error.
>
> ⚠️ **No peguéis notas dentro de los bloques `<<EOF`.** Todo lo que haya entre `EOF` y `EOF` acaba escrito en el fichero.

---

## 1. IP fija

> **🎯 Qué hacemos:** asegurar que el servidor tiene siempre la misma IP. Los clientes la usarán como DNS y los nombres apuntan a ella: si cambia, nada funciona.
> **📄 Archivo:** `/etc/netplan/*.yaml` · se **modifica** (con copia `.bak`) · configuración de red del sistema.

Primero, ver si ya la tiene:
```bash
ip -4 -br a
```
✅ Si sale `<IP>/<máscara>` en la interfaz, pasad al apartado 2.

Si no, configurarla con **netplan** (Ubuntu Server):
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
`netplan try` aplica el cambio y, si no se confirma con Enter en 120 s, **lo revierte solo** (útil por SSH).

> Con **NetworkManager** (típico de Ubuntu Desktop) el equivalente es:
> `sudo nmcli con mod "<conexión>" ipv4.method manual ipv4.addresses ${IP}/${MASCARA} ipv4.gateway ${GW} ipv4.dns "8.8.8.8"` y luego `sudo nmcli con up "<conexión>"`. Los nombres salen de `nmcli con show`.

Comprobar, también tras reiniciar:
```bash
ip -4 -br a
ip route | grep default
ping -c 2 ${GW}
ping -c 2 8.8.8.8
```
✅ Se ve la IP fija, una línea `default via <GW>` y respuesta a los dos pings. (Una segunda dirección por DHCP no afecta.)

> Si responde la puerta de enlace pero no `8.8.8.8`, o ninguna de las dos, **la puerta de enlace es incorrecta**. Debe ser la del router de esa red (`ipconfig` en un Windows de la misma red). Con una errónea, el servidor llega a la red local pero no a internet.

## 2. Instalar BIND9

> **🎯 Qué hacemos:** instalar el servidor DNS.
> **📄 Archivos:** crea la carpeta `/etc/bind/` con la configuración de ejemplo. De momento no se toca ninguno.

```bash
sudo apt update && sudo apt install -y bind9 bind9-utils dnsutils
sudo systemctl status named --no-pager
```
✅ `active (running)`. En Ubuntu el servicio se llama `named`; `bind9` es solo un alias.

## 3. Opciones generales de BIND

> **🎯 Qué hacemos:** decirle a BIND que, si le preguntan por algo que no es nuestro dominio (p. ej. google.com), lo consulte a un DNS público. Así el servidor sigue dando internet.
> **📄 Archivo:** `/etc/bind/named.conf.options` · se **sustituye** (con copia `.bak`) · opciones generales de BIND.

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

- `forwarders` + `forward only`: lo que no es nuestro se pregunta a Google.
- `dnssec-validation no`: en algunas redes las consultas a los servidores raíz fallan (`FORMERR`) y BIND rechaza hasta respuestas correctas (`SERVFAIL`). El DNS público ya valida DNSSEC por su cuenta.

## 4. Crear la zona

> **🎯 Qué hacemos:** crear nuestro dominio (`<ZONA>`) y la lista de nombres que contiene. Son dos ficheros: uno **declara** la zona y otro la **rellena**.
> **📄 Archivos:**
> - `/etc/bind/named.conf.local` · se **sustituye** (con copia `.bak`) · declara que este servidor gestiona `<ZONA>`.
> - `/etc/bind/zones/db.<ZONA>` · se **crea** (la carpeta `zones` también) · lista de dominios: una línea por cada uno.

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
sudo mkdir -p /etc/bind/zones

sudo tee /etc/bind/named.conf.local > /dev/null <<EOF
// ===== ${ZONA} =====
zone "${ZONA}" {
        type primary;
        file "/etc/bind/zones/db.${ZONA}";
};
EOF

sudo tee /etc/bind/zones/db.${ZONA} > /dev/null <<EOF
\$TTL 300
@     IN  SOA ns.${ZONA}. admin.${ZONA}. (
          1 604800 86400 2419200 604800 )
@     IN  NS  ns.${ZONA}.
ns    IN  A   ${IP}
${H1} IN  A   ${IP}
${H2} IN  A   ${IP}
EOF
```

Las cuatro primeras líneas del fichero de zona son "burocracia" del servidor (`SOA`, `NS`, `ns`).

> **`$TTL 300`** son los segundos que los clientes recuerdan una respuesta. En un laboratorio conviene un valor bajo (5 minutos): si cambia una IP, los equipos lo ven enseguida. Con el valor por defecto de BIND (`604800`, siete días) un cliente puede seguir yendo a la IP antigua durante días.

**Lo que importa son las líneas `nombre IN A ip`**: cada una es un dominio. La barra de `\$TTL` evita que la terminal lo tome por una variable.

Comprobar y aplicar:
```bash
sudo named-checkconf
sudo named-checkzone ${ZONA} /etc/bind/zones/db.${ZONA}
sudo systemctl restart named
```
✅ `named-checkconf` no muestra nada y `named-checkzone` termina en `OK`.

## 5. Comprobar en el servidor

> **🎯 Qué hacemos:** verificar que el DNS responde por nuestros nombres y también por los de internet.
> **📄 Archivos:** ninguno, solo consultas.

```bash
dig @${IP} ${H1}.${ZONA} +short
dig @${IP} ${H2}.${ZONA} +short
dig @127.0.0.1 www.google.com | grep -E "status|^www"
sudo systemctl is-enabled named
```
✅ Los dos primeros devuelven la IP del servidor. El tercero, `status: NOERROR` y las IP de Google. El último, `enabled` (arranca solo tras reiniciar).

## 6. Cortafuegos

> **🎯 Qué hacemos:** asegurarnos de que el cortafuegos no bloquea el DNS.
> **📄 Archivos:** ninguno.

```bash
sudo ufw status
```
✅ Si está `inactivo`, no hay que hacer nada. Si estuviera activo:
```bash
sudo ufw allow 53
sudo ufw allow 80/tcp
```

## 7. Acceso desde otro equipo

> **🎯 Qué hacemos:** que otro equipo use nuestro DNS para entrar a las webs por nombre.
> **📄 Archivos:** ninguno en el servidor; se cambia el DNS en el equipo cliente.

1. El equipo debe poder **llegar a la IP del servidor** (misma red o con ruta).
2. Poner la IP del servidor como **servidor DNS** del equipo (adaptador de red → IPv4 → DNS preferido). Alternativas: que lo reparta el DHCP de la red, o que lo delegue otro DNS.
3. En el navegador, **con `http://` delante**: `http://<NOMBRE>.<ZONA>`

**Cliente Windows** (PowerShell como administrador; `Ethernet` es el nombre del adaptador, se ve con `Get-NetAdapter`):

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses <IP_DNS>
Clear-DnsClientCache
```

Para volver al DNS anterior:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
```

Sin cambiar nada en el cliente, se prueba solo la resolución indicando el servidor:
```powershell
nslookup <NOMBRE>.<ZONA> <IP_DNS>
```
✅ Responde con la IP del dominio. (El aviso "No se encuentra el nombre del servidor" es normal: no hay zona inversa.)

---

## 8. Operaciones habituales

> **🎯 Qué hacemos:** añadir, consultar, cambiar y eliminar dominios en el día a día.
> **📄 Archivo:** `/etc/bind/zones/db.<ZONA>` · se **modifica** (ya existe desde el apartado 4) · es la lista de dominios.

### Cómo funciona (leer primero)

Cada dominio es **una línea** de ese fichero:

```text
nuevo     IN  A   192.168.1.20
```

| Parte | Significa |
|---|---|
| `nuevo` | Nombre del dominio. Con la zona `grupo.test`, el completo es `nuevo.grupo.test` |
| `IN  A` | Tal cual, siempre igual |
| `192.168.1.20` | IP del equipo **donde está la web** |

Hay **dos IP distintas** que no hay que confundir:
- **IP del servidor DNS**: la de este equipo. Es la que ponen los clientes como DNS.
- **IP de la web**: la del equipo que sirve la página. Es la que va en la línea (será la misma que la del DNS solo si la web está en este mismo servidor).

> ⚠️ La línea debe empezar **pegada al margen izquierdo**. Si empieza con espacio, significa "mismo nombre que la anterior" y rompe la zona.

En los comandos de abajo, sustituir lo que va entre `< >`:

| Poner... | Ejemplo |
|---|---|
| `<ZONA>` | `grupo.test` |
| `<NOMBRE>` | `nuevo` |
| `<IP_WEB>` | `192.168.1.20` |
| `<IP_DNS>` | `192.168.1.10` (IP de este servidor) |

### 8.0 Aplicar los cambios (siempre al final)

> **🎯 Qué hacemos:** que BIND lea el fichero modificado. Sin esto, los cambios no se notan.

```bash
sudo named-checkzone <ZONA> /etc/bind/zones/db.<ZONA>
sudo systemctl restart named
```
✅ `named-checkzone` debe terminar en `OK`. Si da error, indica la línea mal escrita: corregirla antes de reiniciar.

### 8.1 Añadir un dominio

**Paso 1.** Abrir el fichero:
```bash
sudo nano /etc/bind/zones/db.<ZONA>
```
**Paso 2.** Ir al final y añadir una línea nueva:
```text
<NOMBRE>     IN  A   <IP_WEB>
```
**Paso 3.** Guardar y salir: `Ctrl+O`, `Enter`, `Ctrl+X`.

**Paso 4.** Aplicar (8.0) y comprobar:
```bash
dig @<IP_DNS> <NOMBRE>.<ZONA> +short
```
✅ Devuelve `<IP_WEB>`.

*Ejemplo:* añadir `nuevo.grupo.test` → `192.168.1.20`, con el DNS en `192.168.1.10`.

1. Abrir con `sudo nano /etc/bind/zones/db.grupo.test` y añadir al final:
   ```text
   nuevo     IN  A   192.168.1.20
   ```
2. Guardar, salir y ejecutar:
   ```bash
   sudo named-checkzone grupo.test /etc/bind/zones/db.grupo.test
   sudo systemctl restart named
   dig @192.168.1.10 nuevo.grupo.test +short
   ```

*Alternativa sin editor (una sola vez):*
```bash
echo "<NOMBRE> IN A <IP_WEB>" | sudo tee -a /etc/bind/zones/db.<ZONA>
```

**Comprobar la web** (opcional):
```bash
curl -H "Host: <NOMBRE>.<ZONA>" http://<IP_WEB>
```
- Si su web usa otro puerto, añadir `:<puerto>`. El DNS **no** conoce puertos: van en la URL.
- Si `dig` va bien pero `curl` falla, el problema está en el servidor web de esa máquina, no en este DNS.

### 8.2 Consultar los dominios

Ver el fichero entero (cada línea `nombre IN A ip` es un dominio; `SOA`, `NS` y `ns` son del propio servidor):
```bash
cat /etc/bind/zones/db.<ZONA>
```

Solo nombre completo e IP, alineado:
```bash
awk -v z="<ZONA>" '$2=="IN" && $3=="A" && $1!="ns" {print $1"."z, $4}' /etc/bind/zones/db.<ZONA> | column -t
```

Un dominio concreto, tal como lo responde el DNS:
```bash
dig @<IP_DNS> <NOMBRE>.<ZONA> +short
```

Zonas declaradas en el servidor:
```bash
grep -E '^zone|^//' /etc/bind/named.conf.local
```

### 8.3 Cambiar la IP o el nombre de un dominio

1. `sudo nano /etc/bind/zones/db.<ZONA>`
2. Localizar su línea y **editarla**: la IP, el nombre, o ambos.
3. Guardar y aplicar (8.0).
4. Comprobar con `dig`. Si se cambió el nombre, el antiguo ya no debe resolver.

Si la web está en Nginx, al renombrar cambiar también su `server_name` y recargar: `sudo nginx -t && sudo systemctl reload nginx`.

### 8.4 Eliminar un dominio

1. `sudo nano /etc/bind/zones/db.<ZONA>`
2. Borrar **la línea entera** (en nano, `Ctrl+K` borra la línea del cursor).
3. Guardar y aplicar (8.0).
4. Comprobar: `dig @<IP_DNS> <NOMBRE>.<ZONA> +short` no devuelve nada.

*Alternativa sin editor:*
```bash
sudo sed -i "/^<NOMBRE>[[:space:]]/d" /etc/bind/zones/db.<ZONA>
```

### 8.5 Copia de seguridad y vuelta atrás

Antes de cambios grandes:
```bash
sudo cp /etc/bind/zones/db.<ZONA> /etc/bind/zones/db.<ZONA>.bak
```
Para volver atrás:
```bash
sudo cp /etc/bind/zones/db.<ZONA>.bak /etc/bind/zones/db.<ZONA>
sudo systemctl restart named
```

### 8.6 Varias zonas (grupos con dominio raíz distinto)

> **📄 Archivos:** `named.conf.local` se **modifica** (declarar la zona) y se **crea** un `db.<zona>` nuevo por cada una.

Cada zona se declara en `named.conf.local`, con un comentario de cabecera por grupo:

```text
// ===== GRUPO A =====
zone "a.test" {
        type primary;
        file "/etc/bind/zones/db.a.test";
};

// ===== GRUPO B =====
zone "b.test" {
        type primary;
        file "/etc/bind/zones/db.b.test";
};
```

Pasos: copia de seguridad de `named.conf.local`, `sudo nano /etc/bind/named.conf.local`, pegar el bloque bajo su cabecera (mejor que `tee -a`: ejecutado dos veces duplica la zona), crear el fichero de zona (copiar uno existente y cambiar la zona en las líneas `SOA` y `NS`) y `sudo named-checkconf` antes de reiniciar.

---

## 9. Web de prueba con Nginx (opcional)

> **🎯 Qué hacemos:** montar una página mínima que responda a **cualquier** nombre de la zona e indique con cuál se la ha llamado. Sirve para comprobar que cada nombre llega al servidor.
> **📄 Archivo:** `/etc/nginx/sites-available/prueba` · se **crea** · define la web de prueba. Se activa con un enlace en `sites-enabled`.

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
✅ `nginx -t`: **syntax is ok** y **test is successful**.

Comprobar:
```bash
curl -H "Host: ${H1}.${ZONA}" http://127.0.0.1
curl -H "Host: ${H2}.${ZONA}" http://127.0.0.1
```
✅ Cada uno responde con su propio nombre. (Sin `curl`: `sudo apt install -y curl`.) Desde otro equipo: `http://<NOMBRE>.<ZONA>`.

> **En el propio servidor, `curl http://<NOMBRE>.<ZONA>` puede no devolver nada**, porque el servidor usa su DNS del sistema (por ejemplo `8.8.8.8`) y no el que acaba de montar. Para probar sin depender de eso, usar `-H "Host: …"` como arriba, o forzar la resolución solo para ese comando: `curl -sS --resolve <NOMBRE>.<ZONA>:80:${IP} http://<NOMBRE>.<ZONA>`. Quitar `-s` o usar `-sS` para ver el error si lo hay.

Para una web real con su propio sitio: crear otro fichero en `sites-available` con `server_name <NOMBRE>.<ZONA>;` y su `root`, y enlazarlo en `sites-enabled` como arriba.

---

## 10. Problemas típicos

| Síntoma | Causa típica |
|---|---|
| Ficheros con huecos (`ns.. `, rutas vacías) | Variables sin definir: repetir el apartado 0 |
| `dig` a internet da `SERVFAIL` | Validación DNSSEC activa: `dnssec-validation no;` y `forward only;` (apartado 3) |
| `is-enabled bind9` responde `alias` | Normal: el servicio real es `named` |
| El nombre no resuelve en el cliente | El cliente no usa la IP del servidor como DNS |
| `dig @IP nombre` no devuelve nada | Falta la línea en `zones/db.<ZONA>`, o no se reinició `named` |
| Dominio nuevo no resuelve | Se editó `named.conf.options` en vez del fichero de zona, o falta reiniciar `named` |
| `named-checkzone` da error al añadir un dominio | Línea mal escrita, con espacio delante, o nombre duplicado |
| `named-checkconf` falla tras añadir una zona | Zona declarada dos veces en `named.conf.local` |
| `dig ... AXFR` da `Transfer failed` | Normal: BIND restringe las transferencias de zona. Usar el listado de 8.2 |
| El navegador busca en Google en vez de abrir la web | Falta `http://` delante |
| El nombre resuelve, pero la web no carga | Fallo en el servidor web de esa IP, no en el DNS (8.1) |
| Sale "Welcome to nginx" en la prueba | No se borró el sitio `default` (apartado 9) |
| `nginx -t` da una directiva desconocida | Texto suelto pegado dentro del fichero del sitio |
| `nslookup` sin servidor indicado da error en Windows | Usa el DNS del adaptador, no el nuestro: indicar la IP del servidor al final |
| Tras un reinicio no funciona | IP sin fijar (apartado 1) o variables sin redefinir en la terminal |
| El servidor no tiene internet tras fijar la IP | Puerta de enlace incorrecta: `ip route \| grep default` y `ping -c 2 ${GW}` (apartado 1) |
| Un cliente sigue viendo la web antigua tras cambiar una IP | Caché de DNS: en Windows `Clear-DnsClientCache`; en el navegador `Ctrl+F5` o ventana privada. Con TTL bajo (apartado 4) dura poco |
| `curl http://<NOMBRE>.<ZONA>` no devuelve nada en el servidor | El servidor no usa su propio DNS: usar `curl -sS -H "Host: …" http://<IP>` (apartado 9) |
| El cliente resuelve la IP antigua aunque el DNS ya está corregido | El cliente tiene otro DNS delante del nuestro: comprobar con `nslookup <NOMBRE>.<ZONA> <IP_DNS>` y con `Get-DnsClientServerAddress` |

## 11. Deshacer (opcional)

```bash
sudo cp /etc/bind/named.conf.options.bak /etc/bind/named.conf.options
sudo cp /etc/bind/named.conf.local.bak /etc/bind/named.conf.local
sudo rm -f /etc/bind/zones/db.${ZONA}
sudo systemctl restart named
```
Quitar BIND por completo: `sudo apt purge -y bind9 bind9-utils`.
Quitar la web de prueba: `sudo rm -f /etc/nginx/sites-enabled/prueba /etc/nginx/sites-available/prueba && sudo systemctl reload nginx`.
