# Instalación y configuración de un servidor web con Nginx (Ubuntu)

> Antes de instalar: comprobar que no hay otro servicio escuchando en los puertos que se van a usar (80, 8080 y 443).
>
> ```
> sudo ss -tulpn | grep -E ':(80|8080|443)\s'
> ```
>
> Si no sale nada, los puertos están libres.

## Índice

- [Conceptos](#conceptos)
- [1. Instalar](#1-instalar)
- [2. Crear las webs](#2-crear-las-webs)
- [3. Crear los virtual hosts](#3-crear-los-virtual-hosts)
- [4. Activar los sitios y quitar el de por defecto](#4-activar-los-sitios-y-quitar-el-de-por-defecto)
- [5. Validar y aplicar](#5-validar-y-aplicar)
- [6. Configurar para autoarranque](#6-configurar-para-autoarranque)
- [7. Firewall (opcional)](#7-firewall-opcional)
- [8. Probar](#8-probar)
  - [Probar desde un cliente](#probar-desde-un-cliente)
- [9. Dos webs en el mismo puerto, distinguidas por nombre](#9-dos-webs-en-el-mismo-puerto-distinguidas-por-nombre)
- [10. HTTPS con certificado autofirmado (opcional)](#10-https-con-certificado-autofirmado-opcional)
- [Operaciones habituales](#operaciones-habituales)
  - [Añadir una web nueva](#añadir-una-web-nueva)
  - [Desactivar o eliminar una web](#desactivar-o-eliminar-una-web)
  - [Cambiar el puerto de una web](#cambiar-el-puerto-de-una-web)
  - [Cambiar la IP o la red del servidor](#cambiar-la-ip-o-la-red-del-servidor)
  - [Ver los registros (logs)](#ver-los-registros-logs)
- [Problemas típicos](#problemas-típicos)

## Conceptos

Un **servidor web** recibe peticiones HTTP (*HyperText Transfer Protocol*) o HTTPS de los navegadores y devuelve páginas. En esta práctica se usa **Nginx**.

| Concepto | Qué es | En esta práctica |
|---|---|---|
| Virtual host (`server`) | Bloque de configuración que define **una web**. Un mismo servidor puede tener varias | `sitio80`, `sitio8080` |
| `listen` | Puerto por el que escucha esa web | `80`, `8080`, `443` |
| `server_name` | Nombre (o IP) con el que se llama a esa web. Cuando varias webs comparten puerto, Nginx elige la web según este nombre | `www.jordi.asir2.iespoveda.local` |
| `root` | Carpeta del disco donde están los ficheros de la web (*document root*) | `/var/www/sitio80` |
| `index` | Fichero que se sirve cuando se pide la carpeta sin nombre de fichero | `index.html` |

Puertos habituales: **80** HTTP, **443** HTTPS.

| Carpeta | Para qué sirve |
|---|---|
| `/etc/nginx/sites-available/` | Aquí se **escribe** la configuración de cada web |
| `/etc/nginx/sites-enabled/` | Aquí se **activa**: contiene enlaces simbólicos a los ficheros de `sites-available`. Solo se lee lo que está enlazado aquí |
| `/var/www/` | Ficheros de las webs |
| `/var/log/nginx/` | Registros (`access.log` y `error.log`) |

Una web **no** se distingue solo por el nombre: DNS convierte el nombre en una IP, pero el **puerto** lo decide quien escribe la dirección en el navegador (`http://servidor:8080`) y Nginx lo recoge con `listen`.

## 1. Instalar

```bash
sudo apt update
sudo apt install -y nginx
```

Al instalarse, Nginx arranca solo y sirve una página de bienvenida en el puerto 80 (`Welcome to nginx!`). Es el sitio `default`, que se quitará en el paso 4.

## 2. Crear las webs

Se crean dos carpetas, una por web, cada una con su página.

```bash
sudo mkdir -p /var/www/sitio80 /var/www/sitio8080
```

Web del puerto 80:

```bash
sudo tee /var/www/sitio80/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>Sitio Puerto 80</title></head>
<body><h1>Bienvenido: Puerto 80</h1></body>
</html>
EOF
```

Web del puerto 8080 (copia de la anterior cambiando el texto):

```bash
sudo cp /var/www/sitio80/index.html /var/www/sitio8080/index.html
sudo sed -i 's/Puerto 80/Puerto 8080/g' /var/www/sitio8080/index.html
```

✅ Comprobar el cambio:

```bash
grep '<h1>' /var/www/sitio80/index.html /var/www/sitio8080/index.html
```

Debe salir `Puerto 80` en un fichero y `Puerto 8080` en el otro.

## 3. Crear los virtual hosts

Un fichero por web en `sites-available`. `server_name` lleva la **IP** o el **dominio** que corresponda; aquí, la IP del servidor.

Web del puerto 80:

```bash
sudo tee /etc/nginx/sites-available/sitio80 > /dev/null <<'EOF'
server {
    listen 80;
    server_name 172.16.6.20;
    root /var/www/sitio80;
    index index.html;
}
EOF
```

Web del puerto 8080:

```bash
sudo tee /etc/nginx/sites-available/sitio8080 > /dev/null <<'EOF'
server {
    listen 8080;
    server_name 172.16.6.20;
    root /var/www/sitio8080;
    index index.html;
}
EOF
```

Cada línea termina en `;` y cada bloque se cierra con `}`. Un `;` olvidado es el error más habitual.

## 4. Activar los sitios y quitar el de por defecto

```bash
sudo ln -s /etc/nginx/sites-available/sitio80 /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/sitio8080 /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

`rm` solo borra el **enlace**; el original sigue en `sites-available` por si se necesita.

## 5. Validar y aplicar

```bash
sudo nginx -t
```

✅ Debe decir **syntax is ok** y **test is successful**. Si no, no continuar: leer el error, que indica el fichero y la línea.

Con la validación correcta, aplicar los cambios sin cortar las conexiones activas:

```bash
sudo systemctl reload nginx
```

## 6. Configurar para autoarranque

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

`enable` hace que arranque con el sistema; `--now` además lo arranca ya. En `status` debe aparecer `active (running)`.

## 7. Firewall (opcional)

Solo si `ufw` está activo (`sudo ufw status`):

```bash
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
```

## 8. Probar

En el propio servidor:

```bash
curl http://172.16.6.20
curl http://172.16.6.20:8080
```

✅ Cada petición debe devolver su página: `Puerto 80` en la primera y `Puerto 8080` en la segunda.

Comprobar además que Nginx escucha en los dos puertos:

```bash
sudo ss -tulpn | grep nginx
```

### Probar desde un cliente

Desde el navegador de otro equipo de la red: `http://172.16.6.20` y `http://172.16.6.20:8080`. Si el DNS de la práctica anterior está montado, también `http://www.jordi.asir2.iespoveda.local` (ver `Unit3_dns.md`, apartado 10).

## 9. Dos webs en el mismo puerto, distinguidas por nombre

Hasta ahora cada web tenía su puerto. Con **virtual hosts por nombre**, dos webs comparten el puerto 80 y Nginx decide cuál servir según el nombre que escribió el cliente (cabecera `Host`). Para que cada nombre llegue al servidor hace falta una línea `A` por nombre en la zona DNS (ver `Unit3_dns.md`, apartado 10.1).

Webs de ejemplo, una por nombre:

```bash
sudo mkdir -p /var/www/www /var/www/web2
echo '<h1>www.jordi.asir2.iespoveda.local</h1>' | sudo tee /var/www/www/index.html > /dev/null
echo '<h1>web2.jordi.asir2.iespoveda.local</h1>' | sudo tee /var/www/web2/index.html > /dev/null
```

Un virtual host por nombre, los dos con `listen 80`:

```bash
sudo tee /etc/nginx/sites-available/www > /dev/null <<'EOF'
server {
    listen 80;
    server_name www.jordi.asir2.iespoveda.local;
    root /var/www/www;
    index index.html;
}
EOF

sudo tee /etc/nginx/sites-available/web2 > /dev/null <<'EOF'
server {
    listen 80;
    server_name web2.jordi.asir2.iespoveda.local;
    root /var/www/web2;
    index index.html;
}
EOF
```

Activar y aplicar:

```bash
sudo ln -s /etc/nginx/sites-available/www /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/web2 /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

✅ Comprobar sin depender del DNS (se fuerza la resolución solo para ese comando):

```bash
curl -sS --resolve www.jordi.asir2.iespoveda.local:80:172.16.6.20 http://www.jordi.asir2.iespoveda.local
curl -sS --resolve web2.jordi.asir2.iespoveda.local:80:172.16.6.20 http://web2.jordi.asir2.iespoveda.local
```

Cada nombre debe devolver su propia página, aunque los dos lleguen a la misma IP y al mismo puerto.

> Si se llama al servidor por la IP o por un nombre que ningún `server_name` reconoce, Nginx responde con el **primer** virtual host que tiene ese puerto. `sitio80` (paso 3) y `www` (este apartado) escuchan los dos en el 80: la IP seguirá llevando a `sitio80` porque su `server_name` es la IP, y cada nombre a su web.

## 10. HTTPS con certificado autofirmado (opcional)

HTTPS es HTTP cifrado con TLS (*Transport Layer Security*) y necesita un **certificado**. Uno **autofirmado** lo firma su propio titular, sin una CA (*Certificate Authority*) externa: el cifrado funciona, pero los navegadores muestran un aviso porque no pueden comprobar quién es el titular. Para una web pública se usaría un certificado de una CA reconocida.

Generar la clave privada y el certificado (válido 365 días) con OpenSSL:

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout /etc/nginx/ssl/www.key \
  -out /etc/nginx/ssl/www.crt \
  -subj "/CN=www.jordi.asir2.iespoveda.local"
sudo chmod 600 /etc/nginx/ssl/www.key
```

Durante la generación salen líneas de puntos y signos `+`: es solo el progreso de OpenSSL.

Añadir un virtual host por el puerto 443 que use ese certificado:

```bash
sudo tee /etc/nginx/sites-available/www-https > /dev/null <<'EOF'
server {
    listen 443 ssl;
    server_name www.jordi.asir2.iespoveda.local;
    ssl_certificate     /etc/nginx/ssl/www.crt;
    ssl_certificate_key /etc/nginx/ssl/www.key;
    root /var/www/www;
    index index.html;
}
EOF

sudo ln -s /etc/nginx/sites-available/www-https /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Si `ufw` está activo: `sudo ufw allow 443/tcp`.

✅ Comprobar (`-k` hace que `curl` acepte el certificado sin verificarlo, porque es autofirmado):

```bash
curl -k --resolve www.jordi.asir2.iespoveda.local:443:172.16.6.20 https://www.jordi.asir2.iespoveda.local
```

Debe devolver la página de `www`. Desde un navegador saldrá el aviso de certificado no fiable: es lo esperado con un autofirmado.

Ver los datos del certificado:

```bash
sudo openssl x509 -in /etc/nginx/ssl/www.crt -noout -subject -issuer -dates
```

En un autofirmado, `subject` e `issuer` son **iguales**.

## Operaciones habituales

### Añadir una web nueva

1. Crear su carpeta y su `index.html` en `/var/www/<nombre>`.
2. Crear `/etc/nginx/sites-available/<nombre>` con su `listen`, `server_name` y `root`.
3. Si usa nombre, añadir su línea `A` en la zona DNS (y subir el serial).
4. Activar y aplicar:

```bash
sudo ln -s /etc/nginx/sites-available/<nombre> /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Desactivar o eliminar una web

Desactivar (la configuración se conserva):

```bash
sudo rm /etc/nginx/sites-enabled/<nombre>
sudo nginx -t && sudo systemctl reload nginx
```

Eliminar del todo: además, borrar `/etc/nginx/sites-available/<nombre>` y la carpeta `/var/www/<nombre>`.

### Cambiar el puerto de una web

Editar la línea `listen` de su fichero en `sites-available`, validar y aplicar:

```bash
sudo nano /etc/nginx/sites-available/<nombre>
sudo nginx -t && sudo systemctl reload nginx
```

Si `ufw` está activo, abrir el puerto nuevo y, si ya no se usa, cerrar el antiguo.

### Cambiar la IP o la red del servidor

Las webs que usan la IP en `server_name` hay que actualizarlas. Localizar dónde aparece la IP antigua:

```bash
sudo grep -rn "172\.16\.6\.20" /etc/nginx/sites-available
```

Sustituirla en cada fichero, validar y aplicar:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Las que usan nombre no cambian: se actualiza la línea `A` en la zona DNS.

### Ver los registros (logs)

```bash
sudo tail -n 20 /var/log/nginx/access.log
sudo tail -n 20 /var/log/nginx/error.log
```

`access.log` registra cada petición (quién, qué página y con qué código de respuesta) y `error.log` los fallos. Seguirlos en directo: `sudo tail -f /var/log/nginx/error.log`.

## Problemas típicos

| Síntoma | Causa típica |
|---|---|
| `nginx -t` da error | Falta un `;` o una llave `}`. El mensaje indica fichero y línea |
| Sale `Welcome to nginx!` | No se ha quitado el sitio `default` (paso 4) |
| Funciona en la máquina virtual pero no desde fuera | Firewall (`ufw`) o red de VirtualBox (¿adaptador puente o red interna?) |
| Error `403 Forbidden` | Falta el `index.html` o Nginx no puede leer la carpeta |
| Error `404 Not Found` | Ruta de `root` mal escrita o el fichero no existe |
| `nginx -t` avisa de `conflicting server name ... ignored` | Dos webs con el mismo `listen` y el mismo `server_name`: Nginx ignora la segunda |
| Dos nombres distintos muestran la misma web | Falta el `server_name` correcto en el virtual host, o se está entrando por IP |
| El nombre no resuelve | Falta la línea `A` en la zona DNS o el cliente no usa ese DNS (`dig @172.16.6.20 <nombre>`) |
| `bind() ... failed (98: Address already in use)` | Otro servicio ya usa ese puerto (`sudo ss -tulpn \| grep :<puerto>`) |
| Aviso de certificado en el navegador | Normal con un certificado autofirmado |
