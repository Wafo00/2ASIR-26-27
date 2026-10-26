# Manual: Nginx con 2 virtual hosts en Ubuntu 26.04

**Objetivo:** `http://172.16.5.20` (puerto 80) y `http://172.16.5.20:8080`, cada uno con su web.

## Índice

- [1. Instalar Nginx](#1-instalar-nginx)
- [2. Crear las webs](#2-crear-las-webs)
- [3. Crear los virtual hosts](#3-crear-los-virtual-hosts)
- [4. Activarlos y quitar el sitio por defecto](#4-activarlos-y-quitar-el-sitio-por-defecto)
- [5. Comprobar y aplicar](#5-comprobar-y-aplicar)
- [6. Firewall (solo si UFW está activo)](#6-firewall-solo-si-ufw-está-activo)
- [7. Probar](#7-probar)
- [8. Opcional: acceder por nombre con DNS](#8-opcional-acceder-por-nombre-con-dns)
- [Si algo falla](#si-algo-falla)

## 1. Instalar Nginx

```bash
sudo apt update && sudo apt install -y nginx
```

## 2. Crear las webs

Carpetas, una por web:

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

Web del 8080 (copia de la anterior con retoques):

```bash
sudo cp /var/www/sitio80/index.html /var/www/sitio8080/index.html
sudo sed -i 's/Puerto 80/Puerto 8080/g' /var/www/sitio8080/index.html
```

## 3. Crear los virtual hosts

En `server_name` va la IP o el dominio que corresponda; aquí, la IP.

```bash
sudo tee /etc/nginx/sites-available/sitio80 > /dev/null <<'EOF'
server {
    listen 80;
    server_name 172.16.5.20;
    root /var/www/sitio80;
    index index.html;
}
EOF
```

```bash
sudo tee /etc/nginx/sites-available/sitio8080 > /dev/null <<'EOF'
server {
    listen 8080;
    server_name 172.16.5.20;
    root /var/www/sitio8080;
    index index.html;
}
EOF
```

## 4. Activarlos y quitar el sitio por defecto

```bash
sudo ln -s /etc/nginx/sites-available/sitio80 /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/sitio8080 /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

## 5. Comprobar y aplicar

```bash
sudo nginx -t
```

Debe decir `syntax is ok` y `test is successful`. Si no, no seguir: leer el error, que suele indicar el fichero y la línea.

Con la comprobación correcta, aplicar los cambios:

```bash
sudo systemctl reload nginx
```

## 6. Firewall (solo si UFW está activo)

```bash
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
```

## 7. Probar

```bash
curl http://172.16.5.20
curl http://172.16.5.20:8080
```

También desde el navegador del equipo anfitrión. Debe verse "Puerto 80" en uno y "Puerto 8080" en el otro.

## 8. Opcional: acceder por nombre con DNS

Este apartado **no es necesario** para que las webs funcionen: solo permite entrar con un nombre en lugar de con la IP. Requiere un servidor DNS montado (ver [`SRI/Unit3_dns.md`](../SRI/Unit3_dns.md), apartado 10).

1. En la zona DNS, añadir una línea `A` por cada nombre que apunte a la IP del servidor (por ejemplo `www IN A 172.16.5.20`), subir el serial y recargar la zona.
2. Cambiar `server_name` en el virtual host por ese nombre. Ejemplo para la web del puerto 80:

   ```bash
   sudo sed -i 's/server_name 172.16.5.20;/server_name www.<ZONA>;/' /etc/nginx/sites-available/sitio80
   sudo nginx -t && sudo systemctl reload nginx
   ```

   `<ZONA>` es el dominio de la zona DNS.
3. Probar sin depender del DNS del sistema (se fuerza la resolución solo para ese comando):

   ```bash
   curl --resolve www.<ZONA>:80:172.16.5.20 http://www.<ZONA>
   ```

   Desde otro equipo que use ese DNS: `http://www.<ZONA>`.

## Si algo falla

| Síntoma | Causa típica |
|---|---|
| `nginx -t` da error | Falta un `;` o una llave (el clásico) |
| Sale "Welcome to nginx" | No se borró el `default` (paso 4) |
| Funciona en la máquina virtual pero no desde fuera | Firewall o red de VirtualBox (¿adaptador puente o red interna?) |
| Error 403/404 | Ruta de `root` mal escrita o falta el `index.html` |
| El nombre no resuelve (paso 8) | Falta la línea `A` en la zona o el cliente no usa ese DNS |
