# Manual: Nginx con 2 virtual hosts en Ubuntu 26.04
Objetivo: http://172.16.5.20 (puerto 80) y http://172.16.5.20:8080, cada uno con su web.

## 1. Instalar Nginx
```bash
sudo apt update && sudo apt install -y nginx
```
## 2. Crear las webs
Web del puerto 80:
``` bash
sudo mkdir -p /var/www/sitio80 /var/www/sitio8080
``` 
``` bash
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
sudo cp -r /var/www/sitio80/. /var/www/sitio8080/
```
sudo sed -i 's/Puerto 80/Puerto 8080/g' /var/www/sitio8080/index.html
``` 
## 3. Crear los virtual hosts
``` bash
sudo tee /etc/nginx/sites-available/sitio80 > /dev/null <<'EOF'
server {
    listen 80;
    server_name 172.16.5.20; (ip o dominio que corresponda)
    root /var/www/sitio80;
    index index.html;
}
EOF
```
``` bash
sudo tee /etc/nginx/sites-available/sitio8080 > /dev/null <<'EOF'
server {
    listen 8080;
    server_name 172.16.5.20; (ip o dominio que corresponda)
    root /var/www/sitio8080;
    index index.html;
}
EOF
```
## 4. Activarlos (y quitar el sitio por defecto)
``` bash
sudo ln -s /etc/nginx/sites-available/sitio80 /etc/nginx/sites-enabled/
```
``` bash
sudo ln -s /etc/nginx/sites-available/sitio8080 /etc/nginx/sites-enabled/
```
``` bash
sudo rm /etc/nginx/sites-enabled/default
```
## 5. Comprobar y aplicar
``` bash
sudo nginx -t
```
``` bash
sudo systemctl reload ngin
```
``` bash
nginx -t debe decir syntax is ok y test is successful. Si no, no sigáis: leed el error, que suele indicar el fichero y la línea.
```
## 6. Firewall (solo si UFW está activo)
``` bash
sudo ufw allow 80/tcp
```
``` bash
sudo ufw allow 8080/tcp
```
## 7. Probar
``` bash
curl http://172.16.5.20
```
``` bash
curl http://172.16.5.20:8080
```
También desde el navegador del equipo anfitrión. Debéis ver "Puerto 80" en uno y "Puerto 8080" en el otro.

Si algo falla
Síntoma	Causa típica
nginx -t da error	Falta un ; o una llave (el clásico)
Sale "Welcome to nginx"	No borrasteis el default (paso 4)
Funciona en la MV pero no desde fuera	Firewall o red de VirtualBox (¿adaptador puente o red interna?)
Error 403/404	Ruta de root mal escrita o falta el index.html

