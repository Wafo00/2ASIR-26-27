# Manual de comandos Linux — 2º ASIR

Guía de referencia rápida, organizada **por tareas**: buscar primero *qué se quiere hacer*, no el nombre del comando.

- Con **Ctrl+F** se busca una palabra del "para qué" (p. ej. *puerto*, *permisos*, *clave*).
- Cada entrada numerada trae el comando, sus opciones más usadas y, a veces, un ejemplo.
- En los comandos, `mayúsculas` o palabras como `usuario`, `IP`, `iface`, `fichero` son marcadores: sustituirlos por el valor real.
- Muchos comandos necesitan `sudo`; si la sesión ya es root, se omite.

**Saltar a sección:** [1. Navegación y sistema de archivos](#1-navegación-y-sistema-de-archivos) · [2. Visualización y búsqueda de contenido](#2-visualización-y-búsqueda-de-contenido) · [3. Redirecciones y combinación de comandos](#3-redirecciones-y-combinación-de-comandos) · [4. Usuarios y grupos](#4-usuarios-y-grupos) · [5. Permisos y propietarios](#5-permisos-y-propietarios) · [6. Procesos y recursos](#6-procesos-y-recursos) · [7. Servicios (systemd)](#7-servicios-systemd) · [8. Gestión de paquetes](#8-gestión-de-paquetes) · [9. Red](#9-red) · [10. SSH y acceso remoto](#10-ssh-y-acceso-remoto) · [11. DHCP (Kea)](#11-dhcp-kea) · [12. DNS (BIND9)](#12-dns-bind9) · [13. Servidor web (Nginx)](#13-servidor-web-nginx) · [14. Cifrado y firma (GPG)](#14-cifrado-y-firma-gpg) · [15. Investigar un proceso desconocido](#15-investigar-un-proceso-desconocido) · [16. Windows PowerShell: procesos y servicios](#16-windows-powershell-procesos-y-servicios)

---

## Índice

- **[1. Navegación y sistema de archivos](#1-navegación-y-sistema-de-archivos)**
  - [1.1 Ver el directorio actual](#11-ver-el-directorio-actual) — `pwd`
  - [1.2 Listar el contenido de un directorio](#12-listar-el-contenido-de-un-directorio) — `ls`
  - [1.3 Cambiar de directorio](#13-cambiar-de-directorio) — `cd`
  - [1.4 Crear un directorio](#14-crear-un-directorio) — `mkdir`
  - [1.5 Crear un archivo vacío (o actualizar su fecha)](#15-crear-un-archivo-vacío-o-actualizar-su-fecha) — `touch`
  - [1.6 Copiar archivos o directorios](#16-copiar-archivos-o-directorios) — `cp`
  - [1.7 Copiar el contenido de una carpeta (no la carpeta en sí)](#17-copiar-el-contenido-de-una-carpeta-no-la-carpeta-en-sí) — `cp`
  - [1.8 Crear un enlace simbólico](#18-crear-un-enlace-simbólico) — `ln`
  - [1.9 Mover o renombrar](#19-mover-o-renombrar) — `mv`
  - [1.10 Borrar archivos](#110-borrar-archivos) — `rm`
  - [1.11 Borrar un directorio vacío](#111-borrar-un-directorio-vacío) — `rmdir`
  - [1.12 Limpiar la pantalla](#112-limpiar-la-pantalla) — `clear`
  - [1.13 Ver el historial de comandos](#113-ver-el-historial-de-comandos) — `history`
  - [1.14 Hacer una copia de seguridad antes de editar](#114-hacer-una-copia-de-seguridad-antes-de-editar) — `cp`
  - [1.15 Editar un fichero](#115-editar-un-fichero) — `nano`
  - [1.16 Sustituir texto dentro de un fichero](#116-sustituir-texto-dentro-de-un-fichero) — `sed`
  - [1.17 Consultar la ayuda de un comando](#117-consultar-la-ayuda-de-un-comando) — `man`
  - [1.18 Saber si un comando existe y dónde está instalado](#118-saber-si-un-comando-existe-y-dónde-está-instalado) — `which`
- **[2. Visualización y búsqueda de contenido](#2-visualización-y-búsqueda-de-contenido)**
  - [2.1 Ver todo un archivo de golpe](#21-ver-todo-un-archivo-de-golpe) — `cat`
  - [2.2 Ver un archivo paginado (cómodo en archivos largos)](#22-ver-un-archivo-paginado-cómodo-en-archivos-largos) — `less`
  - [2.3 Ver el principio de un archivo](#23-ver-el-principio-de-un-archivo) — `head`
  - [2.4 Ver el final de un archivo / seguirlo en tiempo real](#24-ver-el-final-de-un-archivo--seguirlo-en-tiempo-real) — `tail`
  - [2.5 Buscar archivos por nombre](#25-buscar-archivos-por-nombre) — `find`
  - [2.6 Buscar texto dentro de archivos](#26-buscar-texto-dentro-de-archivos) — `grep`
  - [2.7 Mostrar texto por pantalla / escribir en un archivo](#27-mostrar-texto-por-pantalla--escribir-en-un-archivo) — `echo`
- **[3. Redirecciones y combinación de comandos](#3-redirecciones-y-combinación-de-comandos)**
  - [3.1 Guardar la salida en un archivo (sobrescribe)](#31-guardar-la-salida-en-un-archivo-sobrescribe) — `comando > archivo`
  - [3.2 Añadir la salida al final de un archivo](#32-añadir-la-salida-al-final-de-un-archivo) — `comando >> archivo`
  - [3.3 Encadenar la salida de un comando como entrada del siguiente](#33-encadenar-la-salida-de-un-comando-como-entrada-del-siguiente) — `comando1 | comando2`
  - [3.4 Escribir varias líneas de un archivo de una vez](#34-escribir-varias-líneas-de-un-archivo-de-una-vez) — `cat << EOF`
- **[4. Usuarios y grupos](#4-usuarios-y-grupos)**
  - [4.1 Saber qué usuario es el actual](#41-saber-qué-usuario-es-el-actual) — `whoami`
  - [4.2 Ver el UID, GID y grupos del usuario actual](#42-ver-el-uid-gid-y-grupos-del-usuario-actual) — `id`
  - [4.3 Ver los grupos del usuario actual](#43-ver-los-grupos-del-usuario-actual) — `groups`
  - [4.4 Crear un usuario](#44-crear-un-usuario) — `adduser`
  - [4.5 Crear un grupo](#45-crear-un-grupo) — `groupadd`
  - [4.6 Añadir un usuario a un grupo (sin quitarle los que ya tenía)](#46-añadir-un-usuario-a-un-grupo-sin-quitarle-los-que-ya-tenía) — `usermod`
  - [4.7 Cambiar el grupo principal de un usuario](#47-cambiar-el-grupo-principal-de-un-usuario) — `usermod`
  - [4.8 Eliminar un usuario (y su directorio personal)](#48-eliminar-un-usuario-y-su-directorio-personal) — `userdel`
  - [4.9 Eliminar un grupo](#49-eliminar-un-grupo) — `groupdel`
  - [4.10 Comprobar si existe un grupo](#410-comprobar-si-existe-un-grupo) — `getent group`
  - [4.11 Ver UID y carpeta personal de uno o varios usuarios](#411-ver-uid-y-carpeta-personal-de-uno-o-varios-usuarios) — `getent passwd`
- **[5. Permisos y propietarios](#5-permisos-y-propietarios)**
  - [5.1 Ver permisos, propietario y grupo](#51-ver-permisos-propietario-y-grupo) — `ls`
  - [5.2 Cambiar permisos (modo simbólico)](#52-cambiar-permisos-modo-simbólico) — `chmod`
  - [5.3 Cambiar permisos (modo numérico: r=4 w=2 x=1)](#53-cambiar-permisos-modo-numérico-r4-w2-x1) — `chmod`
  - [5.4 Cambiar el propietario](#54-cambiar-el-propietario) — `chown`
  - [5.5 Cambiar propietario y grupo a la vez](#55-cambiar-propietario-y-grupo-a-la-vez) — `chown`
  - [5.6 Cambiar solo el grupo](#56-cambiar-solo-el-grupo) — `chown`
  - [5.7 Aplicar cambios de forma recursiva a un directorio](#57-aplicar-cambios-de-forma-recursiva-a-un-directorio) — `chown`
- **[6. Procesos y recursos](#6-procesos-y-recursos)**
  - [6.1 Ver los procesos de la sesión actual](#61-ver-los-procesos-de-la-sesión-actual) — `ps`
  - [6.2 Ver todos los procesos del sistema](#62-ver-todos-los-procesos-del-sistema) — `ps aux`
  - [6.3 Ver procesos eligiendo columnas y ordenados (PID, PPID, usuario, consumo)](#63-ver-procesos-eligiendo-columnas-y-ordenados-pid-ppid-usuario-consumo) — `ps`
  - [6.4 Buscar el PID de un proceso por su nombre](#64-buscar-el-pid-de-un-proceso-por-su-nombre) — `pgrep`
  - [6.5 Ver la jerarquía de procesos (quién es padre de quién)](#65-ver-la-jerarquía-de-procesos-quién-es-padre-de-quién) — `pstree`
  - [6.6 Crear un proceso de prueba y lanzarlo en segundo plano](#66-crear-un-proceso-de-prueba-y-lanzarlo-en-segundo-plano) — `sleep`
  - [6.7 Gestionar trabajos de la terminal (suspender, segundo plano, primer plano)](#67-gestionar-trabajos-de-la-terminal-suspender-segundo-plano-primer-plano) — `jobs`
  - [6.8 Cambiar la prioridad de un proceso (`nice`)](#68-cambiar-la-prioridad-de-un-proceso-nice) — `nice`
  - [6.9 Consultar los datos internos de un proceso (`/proc`)](#69-consultar-los-datos-internos-de-un-proceso-proc) — `cat`
  - [6.10 Monitor de procesos en tiempo real](#610-monitor-de-procesos-en-tiempo-real) — `top`
  - [6.11 Monitor mejorado (si está instalado)](#611-monitor-mejorado-si-está-instalado) — `htop`
  - [6.12 Finalizar un proceso — cierre normal (`SIGTERM`)](#612-finalizar-un-proceso--cierre-normal-sigterm) — `kill`
  - [6.13 Finalizar un proceso — cierre forzado (`SIGKILL`)](#613-finalizar-un-proceso--cierre-forzado-sigkill) — `kill`
  - [6.14 Ver memoria RAM](#614-ver-memoria-ram) — `free`
  - [6.15 Ver espacio en disco](#615-ver-espacio-en-disco) — `df`
  - [6.16 Ver cuánto ocupa un directorio](#616-ver-cuánto-ocupa-un-directorio) — `du`
- **[7. Servicios (systemd)](#7-servicios-systemd)**
  - [7.1 Ver el estado de un servicio](#71-ver-el-estado-de-un-servicio) — `systemctl status`
  - [7.2 Comprobar si un servicio está activo o habilitado](#72-comprobar-si-un-servicio-está-activo-o-habilitado) — `systemctl is-active`
  - [7.3 Ver el fichero de unidad de un servicio](#73-ver-el-fichero-de-unidad-de-un-servicio) — `systemctl cat`
  - [7.4 Ver de qué depende un servicio](#74-ver-de-qué-depende-un-servicio) — `systemctl list-dependencies`
  - [7.5 Gestionar un servicio](#75-gestionar-un-servicio) — `systemctl start|stop|restart|reload`
  - [7.6 Activar el arranque automático de un servicio](#76-activar-el-arranque-automático-de-un-servicio) — `systemctl enable`
  - [7.7 Refrescar unidades tras cambios manuales en disco](#77-refrescar-unidades-tras-cambios-manuales-en-disco) — `systemctl daemon-reload`
  - [7.8 Ver el log de un servicio](#78-ver-el-log-de-un-servicio) — `journalctl`
  - [7.9 Listar los servicios del sistema](#79-listar-los-servicios-del-sistema) — `systemctl`
  - [7.10 Crear el script que hará de servicio](#710-crear-el-script-que-hará-de-servicio) — `nano`
  - [7.11 Crear la unidad `.service`](#711-crear-la-unidad-service) — `nano`
  - [7.12 Poner en marcha el servicio propio](#712-poner-en-marcha-el-servicio-propio) — `systemctl daemon-reload`
  - [7.13 Reparar un servicio que no arranca (ciclo de trabajo)](#713-reparar-un-servicio-que-no-arranca-ciclo-de-trabajo) — `systemctl status`
- **[8. Gestión de paquetes](#8-gestión-de-paquetes)**
  - [8.1 Actualizar el índice de paquetes (no instala nada)](#81-actualizar-el-índice-de-paquetes-no-instala-nada) — `apt update`
  - [8.2 Instalar un programa](#82-instalar-un-programa) — `apt install`
  - [8.3 Actualizar los paquetes instalados](#83-actualizar-los-paquetes-instalados) — `apt upgrade`
  - [8.4 Desinstalar un programa (conserva su configuración)](#84-desinstalar-un-programa-conserva-su-configuración) — `apt remove`
  - [8.5 Desinstalar un programa por completo](#85-desinstalar-un-programa-por-completo) — `apt purge`
  - [8.6 Quitar dependencias que ya no usa nadie](#86-quitar-dependencias-que-ya-no-usa-nadie) — `apt autoremove`
  - [8.7 Evitar que un paquete se quite con `autoremove`](#87-evitar-que-un-paquete-se-quite-con-autoremove) — `apt-mark manual`
  - [8.8 Ver el software instalado](#88-ver-el-software-instalado) — `apt list`
  - [8.9 Comprobar si un paquete concreto está instalado](#89-comprobar-si-un-paquete-concreto-está-instalado) — `dpkg`
  - [8.10 Saber a qué paquete pertenece un fichero del sistema](#810-saber-a-qué-paquete-pertenece-un-fichero-del-sistema) — `dpkg`
  - [8.11 Extraer los ficheros de un paquete sin instalarlo](#811-extraer-los-ficheros-de-un-paquete-sin-instalarlo) — `apt-get download`
- **[9. Red](#9-red)**
  - [9.1 Ver IPs e interfaces de red](#91-ver-ips-e-interfaces-de-red) — `ip a`
  - [9.2 Comprobar si un equipo responde en la red](#92-comprobar-si-un-equipo-responde-en-la-red) — `ping`
  - [9.3 Ver el nombre de la máquina](#93-ver-el-nombre-de-la-máquina) — `hostname`
  - [9.4 Ver IPs de forma compacta (una línea por interfaz)](#94-ver-ips-de-forma-compacta-una-línea-por-interfaz) — `ip`
  - [9.5 Ver la tabla de rutas](#95-ver-la-tabla-de-rutas) — `ip route`
  - [9.6 Ver estadísticas de una interfaz](#96-ver-estadísticas-de-una-interfaz) — `ip`
  - [9.7 Borrar las IPs de una interfaz](#97-borrar-las-ips-de-una-interfaz) — `ip addr`
  - [9.8 Apagar o encender una interfaz](#98-apagar-o-encender-una-interfaz) — `ip link`
  - [9.9 Cambiar la MAC temporalmente](#99-cambiar-la-mac-temporalmente) — `ip link`
  - [9.10 Aplicar la configuración de red (`netplan`)](#910-aplicar-la-configuración-de-red-netplan) — `netplan apply`
  - [9.11 Localizar el fichero de red de `netplan`](#911-localizar-el-fichero-de-red-de-netplan) — `ls`
  - [9.12 Aplicar `netplan` con vuelta atrás automática](#912-aplicar-netplan-con-vuelta-atrás-automática) — `netplan try`
  - [9.13 Elegir por qué interfaz sale a internet (dos tarjetas)](#913-elegir-por-qué-interfaz-sale-a-internet-dos-tarjetas) — `nano`
  - [9.14 Ver las conexiones de `NetworkManager`](#914-ver-las-conexiones-de-networkmanager) — `nmcli connection`
  - [9.15 Poner IP fija y DNS desde la línea de comandos](#915-poner-ip-fija-y-dns-desde-la-línea-de-comandos) — `nmcli connection`
  - [9.16 Renovar IP con `NetworkManager`](#916-renovar-ip-con-networkmanager) — `nmcli device`
  - [9.17 Saber qué servidor DHCP concedió la IP](#917-saber-qué-servidor-dhcp-concedió-la-ip) — `nmcli`
  - [9.18 Liberar y pedir IP manualmente (`dhclient`)](#918-liberar-y-pedir-ip-manualmente-dhclient) — `dhclient`
  - [9.19 Ver el intercambio DHCP en vivo](#919-ver-el-intercambio-dhcp-en-vivo) — `tcpdump`
  - [9.20 Ver qué proceso escucha en un puerto](#920-ver-qué-proceso-escucha-en-un-puerto) — `ss`
  - [9.21 Probar una web desde la terminal](#921-probar-una-web-desde-la-terminal) — `curl`
  - [9.22 Cambiar el nombre de la máquina](#922-cambiar-el-nombre-de-la-máquina) — `hostnamectl set-hostname`
  - [9.23 Ver qué DNS usa el cliente](#923-ver-qué-dns-usa-el-cliente) — `resolvectl status`
  - [9.24 Buscar rastros de una IP antigua en la configuración](#924-buscar-rastros-de-una-ip-antigua-en-la-configuración) — `grep`
  - [9.25 Convertir el servidor en router/NAT](#925-convertir-el-servidor-en-routernat) — `sysctl`
  - [9.26 Ver si el firewall está activo](#926-ver-si-el-firewall-está-activo) — `ufw status`
  - [9.27 Abrir un puerto en el firewall](#927-abrir-un-puerto-en-el-firewall) — `ufw allow`
  - [9.28 Activar el firewall sin perder SSH](#928-activar-el-firewall-sin-perder-ssh) — `ufw allow`
- **[10. SSH y acceso remoto](#10-ssh-y-acceso-remoto)**
  - [10.1 Instalar el servidor SSH y arrancarlo](#101-instalar-el-servidor-ssh-y-arrancarlo) — `apt install`
  - [10.2 Saber si el servicio SSH funciona por socket (Ubuntu reciente)](#102-saber-si-el-servicio-ssh-funciona-por-socket-ubuntu-reciente) — `systemctl status`
  - [10.3 Comprobar si SSH funciona (en este orden)](#103-comprobar-si-ssh-funciona-en-este-orden) — `ping`
  - [10.4 Saber si la sesión es SSH o consola local](#104-saber-si-la-sesión-es-ssh-o-consola-local) — `echo`
  - [10.5 Cerrar una sesión SSH](#105-cerrar-una-sesión-ssh) — `exit`
  - [10.6 Comprobar que SSH escucha en el puerto 22](#106-comprobar-que-ssh-escucha-en-el-puerto-22) — `ss`
  - [10.7 Generar un par de claves](#107-generar-un-par-de-claves) — `ssh-keygen`
  - [10.8 Copiar la clave pública a un servidor](#108-copiar-la-clave-pública-a-un-servidor) — `ssh-copy-id`
  - [10.9 Conectar por SSH](#109-conectar-por-ssh) — `ssh`
  - [10.10 Conectar a una MV con red NAT (reenvío de puertos)](#1010-conectar-a-una-mv-con-red-nat-reenvío-de-puertos) — `ssh`
  - [10.11 Copiar un fichero entre equipos (`scp`)](#1011-copiar-un-fichero-entre-equipos-scp) — `scp`
  - [10.12 Ver la huella de las claves del servidor](#1012-ver-la-huella-de-las-claves-del-servidor) — `ssh-keygen`
  - [10.13 Ver las claves guardadas de un servidor (`known_hosts`)](#1013-ver-las-claves-guardadas-de-un-servidor-known_hosts) — `ssh-keygen`
  - [10.14 Borrar un servidor de `known_hosts` (aviso `REMOTE HOST IDENTIFICATION HAS CHANGED`)](#1014-borrar-un-servidor-de-known_hosts-aviso-remote-host-identification-has-changed) — `ssh-keygen`
  - [10.15 Regenerar las claves SSH del servidor (MV clonada)](#1015-regenerar-las-claves-ssh-del-servidor-mv-clonada) — `rm`
  - [10.16 Añadir una clave pública a un servidor sin `ssh-copy-id`](#1016-añadir-una-clave-pública-a-un-servidor-sin-ssh-copy-id) — `cat`
  - [10.17 Conectar con una clave concreta](#1017-conectar-con-una-clave-concreta) — `ssh`
  - [10.18 Autorizar una clave a otro usuario del servidor](#1018-autorizar-una-clave-a-otro-usuario-del-servidor) — `mkdir`
  - [10.19 Revocar una clave pública (acceso por clave)](#1019-revocar-una-clave-pública-acceso-por-clave) — `nano`
  - [10.20 Entrar y quedar directamente como root](#1020-entrar-y-quedar-directamente-como-root) — `ssh`
  - [10.21 Pasar a root tras conectar](#1021-pasar-a-root-tras-conectar) — `sudo`
  - [10.22 Configurar un alias de conexión](#1022-configurar-un-alias-de-conexión) — `~/.ssh/config`
  - [10.23 Depurar una conexión SSH](#1023-depurar-una-conexión-ssh) — `ssh`
  - [10.24 Permisos que exige SSH para funcionar](#1024-permisos-que-exige-ssh-para-funcionar) — `chmod`
  - [10.25 Ver la configuración efectiva del servidor SSH](#1025-ver-la-configuración-efectiva-del-servidor-ssh) — `sshd`
  - [10.26 Editar `sudoers` de forma segura](#1026-editar-sudoers-de-forma-segura) — `visudo`
- **[11. DHCP (Kea)](#11-dhcp-kea)**
  - [11.1 Validar la configuración sin arrancar el servicio](#111-validar-la-configuración-sin-arrancar-el-servicio) — `kea-dhcp4`
  - [11.2 Ver las concesiones de IP activas](#112-ver-las-concesiones-de-ip-activas) — `cat`
  - [11.3 Depurar bloqueos de AppArmor](#113-depurar-bloqueos-de-apparmor) — `dmesg`
- **[12. DNS (BIND9)](#12-dns-bind9)**
  - [12.1 Instalar el servidor DNS y sus utilidades](#121-instalar-el-servidor-dns-y-sus-utilidades) — `apt install`
  - [12.2 Validar la configuración y las zonas](#122-validar-la-configuración-y-las-zonas) — `named-checkconf`
  - [12.3 Recargar, comprobar estado o vaciar la caché](#123-recargar-comprobar-estado-o-vaciar-la-caché) — `rndc reload`
  - [12.4 Forzar la transferencia de una zona desde el primario](#124-forzar-la-transferencia-de-una-zona-desde-el-primario) — `rndc retransfer`
  - [12.5 Comprobar el arranque automático de BIND9](#125-comprobar-el-arranque-automático-de-bind9) — `systemctl is-enabled`
  - [12.6 Consultar un nombre o una IP (directa/inversa)](#126-consultar-un-nombre-o-una-ip-directainversa) — `dig`
- **[13. Servidor web (Nginx)](#13-servidor-web-nginx)**
  - [13.1 Instalar Nginx](#131-instalar-nginx) — `apt install`
  - [13.2 Comprobar la sintaxis de la configuración](#132-comprobar-la-sintaxis-de-la-configuración) — `nginx`
  - [13.3 Aplicar cambios sin cortar el servicio](#133-aplicar-cambios-sin-cortar-el-servicio) — `systemctl reload`
  - [13.4 Activar un virtual host](#134-activar-un-virtual-host) — `ln`
  - [13.5 Ver en qué puertos escucha](#135-ver-en-qué-puertos-escucha) — `ss`
  - [13.6 Probar el sitio desde la terminal](#136-probar-el-sitio-desde-la-terminal) — `curl`
- **[14. Cifrado y firma (GPG)](#14-cifrado-y-firma-gpg)**
  - [14.1 Crear un par de claves (pública/privada)](#141-crear-un-par-de-claves-públicaprivada) — `gpg --full-generate-key`
  - [14.2 Listar claves](#142-listar-claves) — `gpg --list-keys`
  - [14.3 Exportar la clave pública (para compartirla)](#143-exportar-la-clave-pública-para-compartirla) — `gpg --export`
  - [14.4 Importar la clave pública de otra persona](#144-importar-la-clave-pública-de-otra-persona) — `gpg --import`
  - [14.5 Firmar un fichero](#145-firmar-un-fichero) — `gpg --sign`
  - [14.6 Comprobar que una firma es válida](#146-comprobar-que-una-firma-es-válida) — `gpg --verify`
  - [14.7 Cifrar un fichero para un destinatario](#147-cifrar-un-fichero-para-un-destinatario) — `gpg --encrypt`
  - [14.8 Firmar y cifrar en una sola operación](#148-firmar-y-cifrar-en-una-sola-operación) — `gpg --sign --encrypt`
  - [14.9 Descifrar un fichero](#149-descifrar-un-fichero) — `gpg --decrypt`
  - [14.10 Cifrar con contraseña (sin par de claves)](#1410-cifrar-con-contraseña-sin-par-de-claves) — `gpg --symmetric`
  - [14.11 Generar un certificado de revocación](#1411-generar-un-certificado-de-revocación) — `gpg --gen-revoke`
  - [14.12 Corregir permisos si GPG avisa de `unsafe ownership/permissions`](#1412-corregir-permisos-si-gpg-avisa-de-unsafe-ownershippermissions) — `chmod`
- **[15. Investigar un proceso desconocido](#15-investigar-un-proceso-desconocido)**
  - [15.1 Receta completa: de qué es este proceso](#151-receta-completa-de-qué-es-este-proceso) — `ps`
- **[16. Windows PowerShell: procesos y servicios](#16-windows-powershell-procesos-y-servicios)**
  - [16.1 Listar procesos](#161-listar-procesos) — `Get-Process`
  - [16.2 Abrir un programa y obtener su PID](#162-abrir-un-programa-y-obtener-su-pid) — `Start-Process`
  - [16.3 Finalizar un proceso](#163-finalizar-un-proceso) — `Stop-Process`
  - [16.4 Ver el PID del proceso padre](#164-ver-el-pid-del-proceso-padre) — `Get-CimInstance`
  - [16.5 Consultar el estado de un servicio](#165-consultar-el-estado-de-un-servicio) — `Get-Service`
  - [16.6 Iniciar, parar o reiniciar un servicio](#166-iniciar-parar-o-reiniciar-un-servicio) — `Start-Service`
  - [16.7 Equivalencias entre Linux y PowerShell](#167-equivalencias-entre-linux-y-powershell)

---

## 1. Navegación y sistema de archivos

### 1.1 Ver el directorio actual

```bash
pwd
```

### 1.2 Listar el contenido de un directorio

```bash
ls
```

- `-a` — incluye archivos ocultos
- `-l` — formato detallado (permisos, dueño, tamaño, fecha)
- `-h` — tamaños legibles (se combina con `-l`)
- `-R` — recursivo, entra en subdirectorios
**Ejemplo:** `ls -lah`

### 1.3 Cambiar de directorio

```bash
cd ruta
```

- `cd ~ / cd` — ir al directorio personal
- `cd ..` — subir un nivel
- `cd -` — volver al directorio anterior

### 1.4 Crear un directorio

```bash
mkdir nombre
```

- `-p` — crea también los directorios padre que falten
**Ejemplo:** `mkdir -p ASO/{practicas,copias}`

### 1.5 Crear un archivo vacío (o actualizar su fecha)

```bash
touch archivo
```

### 1.6 Copiar archivos o directorios

```bash
cp origen destino
```

- `-r` — copia directorios de forma recursiva
- `-i` — pregunta antes de sobrescribir
- `-v` — muestra lo que copia

### 1.7 Copiar el contenido de una carpeta (no la carpeta en sí)

```bash
sudo cp -r origen/. destino/
```

### 1.8 Crear un enlace simbólico

```bash
sudo ln -s origen destino
```

**Ejemplo:** `sudo ln -s /etc/nginx/sites-available/x /etc/nginx/sites-enabled/`

### 1.9 Mover o renombrar

```bash
mv origen destino
```

- `-i` — pregunta antes de sobrescribir
- `-v` — muestra lo que hace

### 1.10 Borrar archivos

```bash
rm archivo
```

- `-i` — pide confirmación
- `-r` — borra directorios con todo su contenido
- `-f` — fuerza el borrado sin preguntar
**Ejemplo:** `rm -r temporal` — *revisar la ruta antes de pulsar Enter: no hay papelera*

### 1.11 Borrar un directorio vacío

```bash
rmdir directorio
```

- `-p` — borra también los padres si quedan vacíos

### 1.12 Limpiar la pantalla

```bash
clear
```

- `Ctrl+L` — atajo equivalente

### 1.13 Ver el historial de comandos

```bash
history
```

- `!N` — repite la orden número N
- `!!` — repite la última orden

### 1.14 Hacer una copia de seguridad antes de editar

```bash
sudo cp fichero fichero.bak
```

### 1.15 Editar un fichero

```bash
sudo nano fichero
```

### 1.16 Sustituir texto dentro de un fichero

```bash
sudo sed -i 's/viejo/nuevo/g' fichero
```

### 1.17 Consultar la ayuda de un comando

```bash
man comando
```

- `comando --help` — ayuda rápida

### 1.18 Saber si un comando existe y dónde está instalado

```bash
which comando
```


---

## 2. Visualización y búsqueda de contenido

### 2.1 Ver todo un archivo de golpe

```bash
cat archivo
```

- `-n` — numera las líneas (útil para localizar errores "en la línea N")

### 2.2 Ver un archivo paginado (cómodo en archivos largos)

```bash
less archivo
```

- `q` — salir
- `/palabra` — buscar hacia adelante
- `comando | less` — paginar la salida de cualquier comando (more solo avanza)
**Ejemplo:** `ip a | less`

### 2.3 Ver el principio de un archivo

```bash
head archivo
```

- `-n N` — muestra las primeras N líneas

### 2.4 Ver el final de un archivo / seguirlo en tiempo real

```bash
tail archivo
```

- `-n N` — muestra las últimas N líneas
- `-f` — sigue el archivo en vivo (logs)

### 2.5 Buscar archivos por nombre

```bash
find ruta -name "patrón"
```

- `-type f / -type d` — solo archivos / solo directorios
- `-iname` — sin distinguir mayúsculas
**Ejemplo:** `sudo find / -iname "*.lease*" `2>/dev/null`` — *`2>/dev/null` oculta los errores de permiso*

### 2.6 Buscar texto dentro de archivos

```bash
grep "patrón" archivo
```

- `-i` — ignora mayúsculas/minúsculas
- `-r` — busca de forma recursiva en un directorio
- `-n` — muestra el número de línea
- `-v` — muestra las líneas que NO coinciden
- `-c` — cuenta coincidencias
- `-E "a|b"` — varios patrones a la vez
**Ejemplo:** `grep -rn "texto" carpeta/`

### 2.7 Mostrar texto por pantalla / escribir en un archivo

```bash
echo "texto"
```


---

## 3. Redirecciones y combinación de comandos

### 3.1 Guardar la salida en un archivo (sobrescribe)

```bash
comando > archivo
```

### 3.2 Añadir la salida al final de un archivo

```bash
comando >> archivo
```

### 3.3 Encadenar la salida de un comando como entrada del siguiente

```bash
comando1 | comando2
```

**Ejemplo:** `cat usuarios.txt | grep ventas`

### 3.4 Escribir varias líneas de un archivo de una vez

```bash
cat > archivo << EOF ... EOF
```

- `sudo tee archivo > /dev/null << 'EOF' ... EOF` — igual, pero con `sudo` (evita errores de pegar en `nano`)
- `tee -a` — añade en vez de sobrescribir


---

## 4. Usuarios y grupos

### 4.1 Saber qué usuario es el actual

```bash
whoami
```

### 4.2 Ver el UID, GID y grupos del usuario actual

```bash
id
```

- `id usuario` — igual, de otro usuario

### 4.3 Ver los grupos del usuario actual

```bash
groups
```

### 4.4 Crear un usuario

```bash
sudo adduser usuario
```

### 4.5 Crear un grupo

```bash
sudo groupadd grupo
```

### 4.6 Añadir un usuario a un grupo (sin quitarle los que ya tenía)

```bash
sudo usermod -aG grupo usuario
```

### 4.7 Cambiar el grupo principal de un usuario

```bash
sudo usermod -g grupo usuario
```

### 4.8 Eliminar un usuario (y su directorio personal)

```bash
sudo userdel -r usuario
```

### 4.9 Eliminar un grupo

```bash
sudo groupdel grupo
```

- *falla si es el grupo principal de alguien*

### 4.10 Comprobar si existe un grupo

```bash
getent group grupo
```

### 4.11 Ver UID y carpeta personal de uno o varios usuarios

```bash
getent passwd usuario1 usuario2
```

- *campo 3 = UID, campo 6 = carpeta personal*


---

## 5. Permisos y propietarios

### 5.1 Ver permisos, propietario y grupo

```bash
ls -l
```

**Ejemplo:** `-rw-r--r-- propietario grupo`

### 5.2 Cambiar permisos (modo simbólico)

```bash
chmod u+x,g-w,o= archivo
```

- `u / g / o` — propietario / grupo / otros
- `+ / - / =` — añade / quita / asigna exactamente

### 5.3 Cambiar permisos (modo numérico: r=4 w=2 x=1)

```bash
chmod 755 archivo
```

- `755` — rwxr-xr-x — típico en scripts y directorios
- `644` — rw-r--r-- — típico en archivos normales
- `700` — rwx------ — privado, solo el propietario
- `750` — rwxr-x--- — propietario + grupo, nada para otros

### 5.4 Cambiar el propietario

```bash
sudo chown usuario archivo
```

### 5.5 Cambiar propietario y grupo a la vez

```bash
sudo chown usuario:grupo archivo
```

### 5.6 Cambiar solo el grupo

```bash
sudo chown :grupo archivo
```

- `chgrp grupo archivo` — equivalente

### 5.7 Aplicar cambios de forma recursiva a un directorio

```bash
sudo chown -R usuario:grupo directorio
```


---

## 6. Procesos y recursos

### 6.1 Ver los procesos de la sesión actual

```bash
ps
```

### 6.2 Ver todos los procesos del sistema

```bash
ps aux
```

**Ejemplo:** `ps aux | grep nombre` — *dos líneas: el proceso y el propio `grep`*

### 6.3 Ver procesos eligiendo columnas y ordenados (PID, PPID, usuario, consumo)

```bash
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%mem | head
```

- `-e` — todos los procesos del sistema
- `-o columnas` — elige qué columnas mostrar (`pid`, `ppid` = padre, `user`, `%cpu`, `%mem`, `ni` = nice, `cmd`)
- `--sort=-%mem` — de mayor a menor memoria; `--sort=-%cpu` para CPU (el `-` invierte el orden)
- `| head` — solo las primeras 10 líneas (la cabecera cuenta como una)
**Ejemplo:** `ps -eo pid,ppid,user,cmd | grep nombre` — *dónde está el PPID de un proceso concreto*

### 6.4 Buscar el PID de un proceso por su nombre

```bash
pgrep nombre
```

- `-a` — muestra también la línea de comando completa
- `-x` — nombre exacto (evita que `snapd` coincida con `snapd-desktop-integration`)
- `-n` — solo el proceso más reciente
**Ejemplo:** `pgrep -x snapd` · `kill $(pgrep -n sleep)` — *`$( )` inserta el PID; no escribir la palabra PID literalmente*

### 6.5 Ver la jerarquía de procesos (quién es padre de quién)

```bash
pstree
```

- `-p` — añade el PID de cada proceso
- `pstree -p PID` — solo la rama de ese proceso
- *todo parte del PID 1 (`systemd`); un proceso huérfano pasa a depender de él*

### 6.6 Crear un proceso de prueba y lanzarlo en segundo plano

```bash
sleep 300
sleep 300 &
```

- `sleep N` — espera N segundos; bloquea la terminal mientras dura
- `&` al final — lo lanza en segundo plano y devuelve el control al instante (`[1] PID`)
- `$!` — PID del último proceso lanzado en segundo plano

### 6.7 Gestionar trabajos de la terminal (suspender, segundo plano, primer plano)

```bash
jobs
```

- `Ctrl+Z` — suspende el proceso en primer plano (queda *Stopped*, no muere)
- `bg %1` — reanuda el trabajo 1 en segundo plano
- `fg %1` — trae el trabajo 1 a primer plano
- `Ctrl+C` — cancela el proceso en primer plano
- `kill %1` — termina el trabajo 1 (el número entre `[ ]` de `jobs`)
- *los trabajos pertenecen a esa terminal: al cerrarla, desaparecen*

### 6.8 Cambiar la prioridad de un proceso (`nice`)

```bash
nice -n 10 comando
renice -n 5 -p PID
```

- rango de **-20** (máxima prioridad) a **19** (mínima); por defecto 0
- subir el número = ser más "amable" = menos prioridad; un usuario normal solo puede subirlo
- bajar el número (más prioridad) requiere `sudo`
- `ps -eo pid,ni,cmd` o `top` (columna NI) — comprobar el valor aplicado

### 6.9 Consultar los datos internos de un proceso (`/proc`)

```bash
cat /proc/PID/status
```

- `Name`, `State` (R/S/T/Z), `PPid` (padre), `Uid` (usuario), `VmRSS` (RAM usada)
- `readlink -f /proc/PID/exe` — ruta real del ejecutable
- `readlink /proc/PID/cwd` — directorio desde el que trabaja
- `cat /proc/PID/cmdline` — comando con el que se lanzó
- `cat /proc/PID/cgroup` — a qué servicio o sesión pertenece (`system.slice/servicio.service`)
- *`/proc` es un sistema de archivos virtual: lo genera el kernel en memoria, no ocupa disco*

### 6.10 Monitor de procesos en tiempo real

```bash
top
```

- `q` — salir
- *en español la columna COMMAND se llama ORDEN*

### 6.11 Monitor mejorado (si está instalado)

```bash
htop
```

### 6.12 Finalizar un proceso — cierre normal (`SIGTERM`)

```bash
kill PID
```

- `kill -15 PID` — equivale a `kill PID` (15 = SIGTERM, la señal por defecto)

### 6.13 Finalizar un proceso — cierre forzado (`SIGKILL`)

```bash
kill -9 PID
```

- *usar solo si `kill` normal no responde*
- `kill -9 PID` solo a un proceso propio; los de otros usuarios o de root necesitan `sudo`

### 6.14 Ver memoria RAM

```bash
free -h
```

### 6.15 Ver espacio en disco

```bash
df -h
```

- `df -B M` — en megabytes exactos, sin redondear a GB

### 6.16 Ver cuánto ocupa un directorio

```bash
du -sh directorio
```


---

## 7. Servicios (systemd)

### 7.1 Ver el estado de un servicio

```bash
systemctl status servicio
```

- `--no-pager` — imprime todo sin entrar en `less`
- `systemctl status PID` — también acepta el PID de un proceso para saber de qué servicio es
- *`Active:` indica el estado (`active (running)`, `inactive (dead)`, `failed`); `Loaded:` dice si está `enabled`*

### 7.2 Comprobar si un servicio está activo o habilitado

```bash
systemctl is-active servicio
systemctl is-enabled servicio
```

- `is-active` responde `active`, `inactive` o `failed`
- `is-enabled` responde `enabled` o `disabled`
- *sirven para comprobar rápido, sin leer todo el `status`*

### 7.3 Ver el fichero de unidad de un servicio

```bash
systemctl cat servicio
```

- `systemctl show servicio` — todas las propiedades del servicio (muy larga)
- `systemctl show servicio -p ExecStart` — solo una propiedad
- *los ficheros de unidad propios están en `/etc/systemd/system/`; los de paquetes, en `/usr/lib/systemd/system/`*

### 7.4 Ver de qué depende un servicio

```bash
systemctl list-dependencies servicio | head -20
```

- *la salida es larguísima: recortarla con `head`*
- `systemctl list-dependencies --reverse servicio` — qué depende de él
- *un servicio propio suele colgar de `multi-user.target` → `basic.target` → `sysinit.target`*

### 7.5 Gestionar un servicio

```bash
sudo systemctl start|stop|restart|reload servicio
```

- `reload` — aplica cambios de configuración sin cortar el servicio

### 7.6 Activar el arranque automático de un servicio

```bash
sudo systemctl enable servicio
```

- `systemctl is-enabled servicio` — comprobarlo
- `sudo systemctl enable --now servicio` — habilitarlo y arrancarlo a la vez
- `sudo systemctl disable servicio` — quitar el arranque automático (no lo para si ya está en marcha)
- `sudo systemctl disable --now servicio` — quitar el arranque automático y pararlo

### 7.7 Refrescar unidades tras cambios manuales en disco

```bash
sudo systemctl daemon-reload
```

### 7.8 Ver el log de un servicio

```bash
journalctl -u servicio -e
```

- `-f` — en tiempo real
- `--since "10 min ago"` — desde hace un tiempo concreto
- `-n 20` — solo las últimas 20 líneas
- `-b` — solo desde el último arranque del sistema
- `--no-pager` — sin paginador, útil para capturas

### 7.9 Listar los servicios del sistema

```bash
systemctl --type=service
```

### 7.10 Crear el script que hará de servicio

```bash
#!/bin/bash
while true
do
echo "$(date) - Servicio funcionando"
sleep 10
done
```

- `sudo nano /usr/local/bin/mi-servicio.sh` — crear el fichero con ese contenido (bucle infinito que escribe en el log cada 10 s)
- `sudo chmod +x /usr/local/bin/mi-servicio.sh` — **imprescindible** o aparecerá `203/EXEC`
- `sudo /usr/local/bin/mi-servicio.sh` — probarlo a mano antes de crear el servicio (`Ctrl+C` para salir)

### 7.11 Crear la unidad `.service`

```bash
[Unit]
Description=Servicio de prueba
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mi-servicio.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- `sudo nano /etc/systemd/system/mi-servicio.service` — crear el fichero con ese contenido
- `ExecStart` — ruta **absoluta** del ejecutable
- `Type=simple` — el proceso indicado en `ExecStart` es el propio servicio
- `Restart=on-failure` — se relanza solo si termina con error (un `systemctl stop` manual no lo relanza)
- `WantedBy=multi-user.target` — se arranca en el inicio normal del sistema (necesario para `enable`)

### 7.12 Poner en marcha el servicio propio

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mi-servicio.service
systemctl status mi-servicio.service
```

- `daemon-reload` — obligatorio tras crear o modificar un `.service`
- `journalctl -u mi-servicio.service -n 10 --no-pager` — ver lo que escribe
- `sudo systemctl stop mi-servicio.service` — pararlo; `systemctl is-active` debe decir `inactive`

### 7.13 Reparar un servicio que no arranca (ciclo de trabajo)

```bash
systemctl status servicio
journalctl -u servicio -n 20 --no-pager
sudo nano /etc/systemd/system/servicio.service
sudo systemctl daemon-reload
sudo systemctl reset-failed servicio
sudo systemctl restart servicio
systemctl status servicio
```

- **detectar** → `status` · **consultar** → `journalctl` · **interpretar** el error · **corregir** → editar · **comprobar** → `status`
- `reset-failed` — limpia el contador de fallos si ha saltado el límite de reintentos
- `status=203/EXEC` — `systemd` no pudo ejecutar `ExecStart`: ruta mal escrita o falta `chmod +x`
- `Start request repeated too quickly` / `start-limit-hit` — demasiados fallos seguidos: corregir la causa y luego `reset-failed`
- `Unit ... not found` — nombre mal escrito o falta `daemon-reload`
- `Failed to start ...` — la causa real está en las líneas anteriores del `journalctl`


---

## 8. Gestión de paquetes

### 8.1 Actualizar el índice de paquetes (no instala nada)

```bash
sudo apt update
```

### 8.2 Instalar un programa

```bash
sudo apt install paquete
```

- `-y` — responde sí solo (usar solo si se sabe lo que se instala)

### 8.3 Actualizar los paquetes instalados

```bash
sudo apt upgrade
```

- `apt list --upgradable` — ver qué hay pendiente antes

### 8.4 Desinstalar un programa (conserva su configuración)

```bash
sudo apt remove paquete
```

### 8.5 Desinstalar un programa por completo

```bash
sudo apt purge paquete
```

### 8.6 Quitar dependencias que ya no usa nadie

```bash
sudo apt autoremove
```

- *revisar la lista antes de confirmar*

### 8.7 Evitar que un paquete se quite con `autoremove`

```bash
sudo apt-mark manual paquete
```

- `apt-mark showauto` — ver qué paquetes están marcados como automáticos

### 8.8 Ver el software instalado

```bash
apt list --installed
```

- `dpkg -l` — formato alternativo, más técnico

### 8.9 Comprobar si un paquete concreto está instalado

```bash
dpkg -l | grep paquete
```

### 8.10 Saber a qué paquete pertenece un fichero del sistema

```bash
dpkg -S ruta
```

### 8.11 Extraer los ficheros de un paquete sin instalarlo

```bash
apt-get download paquete
```

- `dpkg -x paquete.deb carpeta` — extraer su contenido (p.ej. recuperar un .conf de fábrica)


---

## 9. Red

### 9.1 Ver IPs e interfaces de red

```bash
ip a
```

### 9.2 Comprobar si un equipo responde en la red

```bash
ping IP
```

- `-c N` — envía solo N paquetes y se detiene
- `Ctrl+C` — detenerlo (muestra el resumen)
- *que responda no implica que sus servicios funcionen (SSH, web...): solo que su red responde*

### 9.3 Ver el nombre de la máquina

```bash
hostname
```

- `hostname -I` — solo sus IPs

### 9.4 Ver IPs de forma compacta (una línea por interfaz)

```bash
ip -br a
```

- `ip -br link` — ver las MAC

### 9.5 Ver la tabla de rutas

```bash
ip route
```

### 9.6 Ver estadísticas de una interfaz

```bash
ip -s -4 addr show iface
```

### 9.7 Borrar las IPs de una interfaz

```bash
sudo ip addr flush dev iface
```

- `sudo ip addr del IP/24 dev iface` — borrar solo una IP concreta

### 9.8 Apagar o encender una interfaz

```bash
sudo ip link set iface down|up
```

### 9.9 Cambiar la MAC temporalmente

```bash
sudo ip link set iface address MAC
```

- *el DHCP lo trata como un cliente nuevo*

### 9.10 Aplicar la configuración de red (`netplan`)

```bash
sudo netplan apply
```

- `--debug` — si falla, para ver el detalle
- *YAML con espacios, nunca tabuladores*

### 9.11 Localizar el fichero de red de `netplan`

```bash
ls /etc/netplan/
```

### 9.12 Aplicar `netplan` con vuelta atrás automática

```bash
sudo netplan try
```

- *si no confirmas, revierte solo a los 120 s: ideal al cambiar la IP por SSH*

### 9.13 Elegir por qué interfaz sale a internet (dos tarjetas)

```bash
sudo nano /etc/netplan/fichero.yaml
```

- `dhcp4-overrides: route-metric: 50` — menor métrica = esa ruta por defecto gana
- `optional: true` — evita esperas al arrancar si falta esa red

### 9.14 Ver las conexiones de `NetworkManager`

```bash
nmcli connection show
```

### 9.15 Poner IP fija y DNS desde la línea de comandos

```bash
nmcli connection modify "conexión" ipv4.method manual ipv4.addresses IP/24 ipv4.dns IP_DNS ipv4.dns-search dominio ipv4.ignore-auto-dns yes
```

- `nmcli connection up "conexión"` — aplicar los cambios

### 9.16 Renovar IP con `NetworkManager`

```bash
nmcli device disconnect iface && nmcli device connect iface
```

### 9.17 Saber qué servidor DHCP concedió la IP

```bash
nmcli -f ALL device show | grep -i dhcp_server_identifier
```

### 9.18 Liberar y pedir IP manualmente (`dhclient`)

```bash
sudo dhclient -r iface && sudo dhclient iface
```

### 9.19 Ver el intercambio DHCP en vivo

```bash
sudo tcpdump -i iface -n port 67 or port 68
```

### 9.20 Ver qué proceso escucha en un puerto

```bash
sudo ss -tulpn | grep ':puerto'
```

- `sudo ss -tlnp | grep nginx` — en qué puertos escucha un servicio concreto

### 9.21 Probar una web desde la terminal

```bash
curl http://ip:puerto
```

- `wget -qO- http://ip:puerto` — alternativa
- `sudo apt install -y curl` — si `curl` no está instalado

### 9.22 Cambiar el nombre de la máquina

```bash
sudo hostnamectl set-hostname nombre
```

- *actualizar también `/etc/hosts`*

### 9.23 Ver qué DNS usa el cliente

```bash
resolvectl status | grep -i 'dns server'
```

### 9.24 Buscar rastros de una IP antigua en la configuración

```bash
sudo grep -rn "IP_antigua" /etc/carpeta
```

- *sin `2>/dev/null`, que ocultaría errores*

### 9.25 Convertir el servidor en router/NAT

```bash
sudo sysctl -p
```

- `net.ipv4.ip_forward=1` — añadirlo antes en `/etc/sysctl.conf`
- `sudo iptables -t nat -A POSTROUTING -o iface_internet -j MASQUERADE` — activar el NAT
- `sudo netfilter-persistent save` — hacerlo permanente

### 9.26 Ver si el firewall está activo

```bash
sudo ufw status verbose
```

- `service --status-all` — no refleja el estado real de UFW: usar `ufw status`

### 9.27 Abrir un puerto en el firewall

```bash
sudo ufw allow puerto
```

- `80/tcp` — ejemplo: abrir el puerto web
- `53 / 67/udp` — DNS / DHCP, según el servicio
- `OpenSSH` — permitir SSH por nombre de perfil

### 9.28 Activar el firewall sin perder SSH

```bash
sudo ufw allow 22/tcp
```

- `sudo ufw enable` — solo después de permitir el 22 (o OpenSSH)


---

## 10. SSH y acceso remoto

### 10.1 Instalar el servidor SSH y arrancarlo

```bash
sudo apt update && sudo apt install -y openssh-server
```

- `sudo systemctl enable --now ssh` — arrancarlo y que inicie con el sistema
- `systemctl status ssh --no-pager` — ver si está activo

### 10.2 Saber si el servicio SSH funciona por socket (Ubuntu reciente)

```bash
systemctl status ssh.socket
```

- *activo (listening) en el 22: `systemd` vigila el puerto y arranca `ssh.service` al recibir una conexión*
- *por eso `ssh.service` puede salir inactive (dead) y SSH funcionar; `ss -tlnp` muestra `systemd` (PID 1), no `sshd`*
- `sudo systemctl stop ssh.socket ssh` — para cortar SSH de verdad (parar solo `ssh.service` no basta: el socket lo reactiva)
- `sudo systemctl start ssh.socket` — volver a dejarlo como estaba

### 10.3 Comprobar si SSH funciona (en este orden)

```bash
ping IP
```

- `ip a` — ¿la IP del servidor es la esperada? (en el servidor)
- `systemctl status ssh` — ¿servicio activo?
- `ps aux | grep sshd` — ¿existe el proceso?
- `sudo ss -tlnp` — ¿puerto 22 en LISTEN?
- `ssh usuario@IP` — ¿usuario y contraseña correctos?

### 10.4 Saber si la sesión es SSH o consola local

```bash
echo $SSH_CONNECTION
```

- *vacío = consola local; con valor = IP/puerto del cliente y del servidor*
- `hostname` — el nombre que sale es el del servidor mientras la sesión esté abierta

### 10.5 Cerrar una sesión SSH

```bash
exit
```

- `Ctrl+D` — equivalente
- *el prompt vuelve al del equipo cliente*

### 10.6 Comprobar que SSH escucha en el puerto 22

```bash
sudo ss -tlnp | grep :22
```

### 10.7 Generar un par de claves

```bash
ssh-keygen -t ed25519 -C "comentario"
```

- `-f ruta/nombre` — nombre propio para el par
- `-t rsa -b 3072` — par RSA de 3072 bits (con Ed25519 no se usa `-b`)
- `ssh-keygen -lf nombre.pub` — ver la huella de una pública
- *sin extensión = privada; `.pub` = pública*

### 10.8 Copiar la clave pública a un servidor

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@IP
```

### 10.9 Conectar por SSH

```bash
ssh usuario@IP
```

- `-p puerto` — si no usa el puerto 22
- `-J salto` — a través de un servidor intermedio
- *la primera vez pregunta si confías en la huella (yes)*

### 10.10 Conectar a una MV con red NAT (reenvío de puertos)

```bash
ssh -p PUERTO_ANFITRION usuario@127.0.0.1
```

- *VirtualBox (MV apagada): Adaptador NAT → Avanzadas → Reenvío de puertos → TCP, IP anfitrión 127.0.0.1, puerto anfitrión → puerto invitado 22*
- *la IP interna de la NAT (10.0.x.x) no es accesible desde el anfitrión: solo se llega por el reenvío*

### 10.11 Copiar un fichero entre equipos (`scp`)

```bash
scp -P puerto usuario@IP:ruta_origen destino_local
```

- *en `scp` el puerto es `-P` mayúscula (en `ssh` es `-p` minúscula)*

### 10.12 Ver la huella de las claves del servidor

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

- `sudo ls -l /etc/ssh/ssh_host_*` — ver las claves de host (privadas `-rw-------`)
- *compárala con la que muestra el cliente en la primera conexión*

### 10.13 Ver las claves guardadas de un servidor (`known_hosts`)

```bash
ssh-keygen -F "[IP]:puerto"
```

- `-lF` — lo mismo, mostrando huellas
- *con puerto no estándar, la IP va entre corchetes*

### 10.14 Borrar un servidor de `known_hosts` (aviso `REMOTE HOST IDENTIFICATION HAS CHANGED`)

```bash
ssh-keygen -R IP
```

- *solo toca el cliente; hazlo tras verificar que el servidor es el correcto*

### 10.15 Regenerar las claves SSH del servidor (MV clonada)

```bash
sudo rm /etc/ssh/ssh_host_*
```

- `sudo dpkg-reconfigure openssh-server` — genera claves nuevas

### 10.16 Añadir una clave pública a un servidor sin `ssh-copy-id`

```bash
cat clave.pub | ssh usuario@IP "cat >> ~/.ssh/authorized_keys"
```

- `>>` — añade sin borrar las que ya hay

### 10.17 Conectar con una clave concreta

```bash
ssh -i ~/.ssh/nombre usuario@IP
```

- `-o PreferredAuthentications=publickey -o PasswordAuthentication=no -o IdentitiesOnly=yes` — forzar solo clave pública

### 10.18 Autorizar una clave a otro usuario del servidor

```bash
sudo mkdir -p /home/otro/.ssh
```

- `grep "comentario" ~/.ssh/authorized_keys | sudo tee /home/otro/.ssh/authorized_keys` — copiar la línea de la clave
- `sudo chown -R otro:otro /home/otro/.ssh` — dueño correcto
- `sudo chmod 700 /home/otro/.ssh && sudo chmod 600 /home/otro/.ssh/authorized_keys` — permisos que exige SSH
- `sudo ls -ld /home/otro/.ssh` — comprobar
- *cada cuenta tiene su propio `authorized_keys`, independiente*

### 10.19 Revocar una clave pública (acceso por clave)

```bash
nano ~/.ssh/authorized_keys
```

- `Ctrl+K / Ctrl+O / Ctrl+X` — cortar línea / guardar / salir (la línea se reconoce por el comentario)
- `sudo sed -i '/comentario/d' ruta/authorized_keys` — borrarla sin abrir el editor
- *el acceso lo controla el servidor: la privada del cliente no hay que borrarla*
- *conectar con la clave revocada da: Permission denied (publickey)*

### 10.20 Entrar y quedar directamente como root

```bash
ssh -tt usuario@IP "sudo -i"
```

- `-tt` — fuerza la asignación de terminal

### 10.21 Pasar a root tras conectar

```bash
sudo -i
```

- `sudo su -` — equivalente

### 10.22 Configurar un alias de conexión

```bash
~/.ssh/config
```

- `Host / HostName / User / IdentityFile` — campos básicos del alias
- `RequestTTY force + RemoteCommand sudo -i` — el alias entra ya como root

### 10.23 Depurar una conexión SSH

```bash
ssh -v usuario@IP
```

- `ssh -G alias` — ver la configuración efectiva de un alias

### 10.24 Permisos que exige SSH para funcionar

```bash
chmod 700 ~/.ssh
```

- `chmod 600 ~/.ssh/authorized_keys` — también necesario

### 10.25 Ver la configuración efectiva del servidor SSH

```bash
sudo sshd -T | grep -i passwordauthentication
```

- `PasswordAuthentication / PermitRootLogin` — se definen en `/etc/ssh/sshd_config` y `sshd_config`.d/

### 10.26 Editar `sudoers` de forma segura

```bash
sudo visudo
```


---

## 11. DHCP (Kea)

### 11.1 Validar la configuración sin arrancar el servicio

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

### 11.2 Ver las concesiones de IP activas

```bash
sudo cat /var/lib/kea/kea-leases4.csv
```

- `column -s, -t fichero | less -S` — verlo en formato tabla

### 11.3 Depurar bloqueos de AppArmor

```bash
sudo dmesg | grep -i apparmor | grep -i kea
```

- `sudo apt install apparmor-utils` — necesario para aa-complain / aa-enforce
- `sudo aa-status` — ver perfiles activos
- `sudo aa-complain / aa-enforce ruta` — modo permisivo / estricto para un binario


---

## 12. DNS (BIND9)

### 12.1 Instalar el servidor DNS y sus utilidades

```bash
sudo apt install bind9 bind9-utils bind9-dnsutils
```

### 12.2 Validar la configuración y las zonas

```bash
sudo named-checkconf
```

- `sudo named-checkzone zona fichero` — validar una zona concreta

### 12.3 Recargar, comprobar estado o vaciar la caché

```bash
sudo rndc reload
```

- `rndc status` — estado
- `rndc flush` — vaciar caché

### 12.4 Forzar la transferencia de una zona desde el primario

```bash
sudo rndc retransfer zona
```

- `sudo rndc zonestatus zona` — estado de una zona
- `dig @primario zona AXFR` — pedir la transferencia completa (solo la permite allow-transfer)

### 12.5 Comprobar el arranque automático de BIND9

```bash
systemctl is-enabled named
```

- *`bind9` es solo un alias de `named`*

### 12.6 Consultar un nombre o una IP (directa/inversa)

```bash
dig @servidor nombre +short
```

- `dig @servidor -x IP` — consulta inversa
- `nslookup nombre servidor` — alternativa


---

## 13. Servidor web (Nginx)

### 13.1 Instalar Nginx

```bash
sudo apt update && sudo apt install -y nginx
```

### 13.2 Comprobar la sintaxis de la configuración

```bash
sudo nginx -t
```

### 13.3 Aplicar cambios sin cortar el servicio

```bash
sudo systemctl reload nginx
```

### 13.4 Activar un virtual host

```bash
sudo ln -s /etc/nginx/sites-available/x /etc/nginx/sites-enabled/
```

### 13.5 Ver en qué puertos escucha

```bash
sudo ss -tlnp | grep nginx
```

### 13.6 Probar el sitio desde la terminal

```bash
curl http://ip:puerto
```


---

## 14. Cifrado y firma (GPG)

### 14.1 Crear un par de claves (pública/privada)

```bash
gpg --full-generate-key
```

- *pide tipo, tamaño, validez, nombre, correo y passphrase*

### 14.2 Listar claves

```bash
gpg --list-keys
```

- `-k` — abreviatura de `--list-keys`
- `gpg --list-secret-keys (-K)` — las privadas disponibles
- `gpg --list-keys --fingerprint` — huella completa: identifica la clave de forma fiable

### 14.3 Exportar la clave pública (para compartirla)

```bash
gpg --export --armor correo > clave_publica.asc
```

- *la privada (`--export-secret-key`) nunca se comparte*

### 14.4 Importar la clave pública de otra persona

```bash
gpg --import clave.asc
```

### 14.5 Firmar un fichero

```bash
gpg --sign fichero
```

- `--detach-sign` — firma en un archivo .sig aparte, sin tocar el original
- *se usa automáticamente la clave privada propia*

### 14.6 Comprobar que una firma es válida

```bash
gpg --verify fichero.sig fichero
```

- *"Firma correcta" si el original no cambió; si se modifica, sale "Firma INCORRECTA" (BAD signature)*

### 14.7 Cifrar un fichero para un destinatario

```bash
gpg --output salida.asc --armor --encrypt --recipient correo fichero
```

- *si avisa de clave no certificada: responde s solo si confías en su origen*
- *si falla con "Host desconocido": reimporta su clave con `gpg --import`*

### 14.8 Firmar y cifrar en una sola operación

```bash
gpg --output entrega.asc --armor --sign --encrypt --recipient correo fichero
```

- *firma con la clave privada propia y cifra con la pública del destinatario*
- *lo firmado+cifrado para otro no se verifica sin descifrar: por eso se usa `--detach-sign` aparte*

### 14.9 Descifrar un fichero

```bash
gpg --decrypt fichero.gpg
```

### 14.10 Cifrar con contraseña (sin par de claves)

```bash
gpg --symmetric fichero
```

- `-c` — abreviatura

### 14.11 Generar un certificado de revocación

```bash
gpg --gen-revoke correo
```

- *por si se compromete la clave privada*

### 14.12 Corregir permisos si GPG avisa de `unsafe ownership/permissions`

```bash
chmod 700 ~/.gnupg
```

- `chown -R $USER:$USER ~/.gnupg` — común en VMs clonadas


---

## 15. Investigar un proceso desconocido

### 15.1 Receta completa: de qué es este proceso

```bash
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%mem | head
pstree -p
readlink -f /proc/PID/exe
systemctl status PID
cat /proc/PID/cgroup
sudo ss -tulpn | grep PID
dpkg -S /ruta/del/ejecutable
```

- **1. Proceso** y **2. Usuario** → primer comando (columnas `pid` y `user`)
- **3. Origen** → columna `ppid` o `pstree -p`: quién lo lanzó
- **4. Ejecutable** → `readlink -f /proc/PID/exe` (la ruta real, no el nombre que aparenta)
- **5. Servicio** → `systemctl status PID` o `cgroup` (`system.slice/xxx.service`; si pone `user.slice`, es de una sesión de usuario)
- **6. Red** → `sudo ss -tulpn | grep PID`: si no hay salida, no escucha en ningún puerto
- **7. Paquete** → `dpkg -S ruta`: a qué paquete pertenece el ejecutable (si no pertenece a ninguno, sospechar)
- **8. Conclusión** → qué es, si es normal y si procede actuar
- *si el proceso ya no existe, las rutas `/proc/PID/...` darán error: anotarlo como "terminó"*


---

## 16. Windows PowerShell: procesos y servicios

### 16.1 Listar procesos

```bash
Get-Process
```

- `Get-Process nombre` — solo los de ese nombre (sin `.exe`)
- `Get-Process -Id PID` — por PID
- *columnas: `CPU(s)` = segundos de CPU, `WS(K)` = memoria en KB, `Id` = PID*

### 16.2 Abrir un programa y obtener su PID

```bash
Start-Process notepad
```

- `Get-Process notepad` — para ver el PID del que acaba de abrirse
- *algunos programas actuales lanzan un proceso "lanzador" que termina enseguida: el que queda es otro*

### 16.3 Finalizar un proceso

```bash
Stop-Process -Id PID
```

- `Stop-Process -Name notepad` — por nombre (cierra **todos** los de ese nombre)
- `Get-Process notepad` — comprobar: debe dar error "no se encuentra ningún proceso"
- *equivale a `kill PID` / `kill -9 PID`*

### 16.4 Ver el PID del proceso padre

```bash
Get-CimInstance Win32_Process -Filter "ProcessId=PID"
```

- *en la salida, el campo `ParentProcessId` es el PPID*
- *si el padre ya terminó, `Get-Process -Id PPID` dará error: es normal y se anota como "no existe"*

### 16.5 Consultar el estado de un servicio

```bash
Get-Service Spooler
```

- `Get-Service` — lista todos los servicios
- `Status` — `Running` o `Stopped`
- `Get-CimInstance Win32_Service -Filter "Name='Spooler'"` — incluye `State` y `StartMode` (`Auto`, `Manual`, `Disabled`)

### 16.6 Iniciar, parar o reiniciar un servicio

```bash
Start-Service Spooler
Stop-Service Spooler
Restart-Service Spooler
```

- *hay que abrir PowerShell **como administrador**; si no, "acceso denegado"*
- `Get-Service Spooler` — comprobar el cambio de estado después de cada orden

### 16.7 Equivalencias entre Linux y PowerShell

```bash
Linux  →  PowerShell
```

- `ps aux` → `Get-Process`
- `kill PID` → `Stop-Process -Id PID`
- `systemctl status servicio` → `Get-Service servicio`
- `systemctl start|stop|restart servicio` → `Start-Service` / `Stop-Service` / `Restart-Service`
- `ps -eo pid,ppid,cmd` → `Get-CimInstance Win32_Process` (`ParentProcessId`)
- `systemctl is-enabled servicio` → `StartMode` en `Win32_Service`
