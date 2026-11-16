# Manual de supervivencia — ASO

Casos de trabajo paso a paso, conceptos generales y referencia rápida.

## Índice

- [1. Cómo usar este manual](#s1)
  - [1.1 Cómo leer los comandos](#e1_1)
  - [1.2 Ciclo de trabajo ante cualquier fallo](#e1_2)
  - [1.3 Comprobaciones y capturas](#e1_3) — `| head -20`
- [2. Casos de trabajo](#s2)
  - [2.1 Tarea bloqueada en la terminal](#e2_1) — `ps -o … -u usuario`
  - [2.2 Investigar un proceso desconocido](#e2_2) — `ps -fp PID`
  - [2.3 Provocar y resolver una incidencia en un servicio](#e2_3) — `journalctl -u servicio`
  - [2.4 Auditar un servicio](#e2_4) — `systemctl cat servicio`
  - [2.5 Servidor lento](#e2_5) — `top`
  - [2.6 Windows: aplicación bloqueada y servicio parado](#e2_6) — `Get-Process -Name nombre`
  - [2.7 Instalar o eliminar un paquete](#e2_7) — `sudo apt remove paquete`
  - [2.8 SSH: conectar y diagnosticar](#e2_8) — `ssh usuario@IP`
  - [2.9 Carpetas, ficheros y datos del equipo](#e2_9) — `mkdir -p`
  - [2.10 Buscar un fichero y filtrar su contenido](#e2_10) — `find … -name`
  - [2.11 Leer y cambiar permisos](#e2_11) — `chmod 640 archivo`
  - [2.12 Carpeta para un departamento](#e2_12) — `chmod 770 carpeta`
- [3. Conceptos generales](#s3)
  - [3.1 Procesos](#e3_1)
  - [3.2 Paquetes](#e3_2)
  - [3.3 Ficheros y búsqueda](#e3_3)
  - [3.4 Usuarios y permisos](#e3_4)
  - [3.5 Red, SSH y Windows](#e3_5)
- [4. Referencia rápida](#s4)
  - [4.1 Mensajes de error de systemd](#e4_1) — `status=203/EXEC…`
  - [4.2 Crear un servicio propio, paso 1: el script](#e4_2) — `chmod +x`
  - [4.3 Crear un servicio propio, paso 2: la unidad](#e4_3) — `nano ….service`
  - [4.4 Crear un servicio propio, paso 3: activarlo](#e4_4) — `enable --now`
  - [4.5 Trabajos en segundo plano](#e4_5) — `Ctrl+Z, jobs, fg, bg`

<a id="s1"></a>

## 1. Cómo usar este manual

<a id="e1_1"></a>

### 1.1 Cómo leer los comandos

Cinco reglas que evitan casi todos los errores.

```
journalctl -u cron -n 20 --no-pager
```
- **Marcadores:** las palabras en minúscula que describen algo (`servicio`, `PID`, `ruta`, `nombre`, `usuario`) se **sustituyen por el valor real**. `systemctl status servicio` se escribe, por ejemplo, `systemctl status cron`.
- **PID:** número que identifica a un proceso. Se saca de `ps` o de `top`.
- **`sudo`** delante: ejecutar como administrador. Hace falta para tocar servicios, ficheros del sistema y procesos de otros usuarios.
- **Opciones** (`-n 20`, `-b`, `--no-pager`): **se añaden** al comando y su orden da igual. El ejemplo de arriba lleva tres: `-u cron`, `-n 20` y `--no-pager`.
- **Salir:** `q` cierra pantallas como `top`, `less` o `journalctl`. `Ctrl+C` corta un comando que no termina.

<a id="e1_2"></a>

### 1.2 Ciclo de trabajo ante cualquier fallo

Siempre el mismo orden. Si algo no se entiende, volver al paso anterior.

```
Detectar → Consultar → Interpretar → Corregir → Comprobar
```
- **Detectar:** ver qué falla (`systemctl status servicio`, `top`, `ps`).
- **Consultar:** leer el registro (`journalctl -u servicio -n 20 --no-pager`).
- **Interpretar:** entender el mensaje de error (apartado 4.1).
- **Corregir:** cambiar solo lo necesario.
- **Comprobar:** repetir el comando de detectar y ver que ya está bien.

<a id="e1_3"></a>

### 1.3 Comprobaciones y capturas

Cada caso marca con «Comprobación» lo que conviene capturar.

```
comando | head -20
comando | tail -20
```
- En cada captura deben verse el **comando**, su **salida** y el prompt.
- Capturar también el error inicial, no solo el resultado final.
- Salidas largas: `| head -20` deja las 20 primeras líneas y `| tail -20` las 20 últimas.
- Cada informe, en este orden: **qué se ve → qué significa → qué se hace → cómo se comprueba**.

<a id="s2"></a>

## 2. Casos de trabajo

<a id="e2_1"></a>

### 2.1 Tarea bloqueada en la terminal

**Situación:** Un usuario lanzó varias tareas desde una terminal y una se ha quedado bloqueada. Localizarla, ver su padre, estado, plano y prioridad, y finalizarla.

**1 · Ver los procesos del usuario**

```
ps -o pid,ppid,user,stat,ni,cmd -u usuario
```
- `-o` → elige las columnas. `-u usuario` → solo los procesos de ese usuario.
- Cada línea es un proceso: `PID`, `PPID` (padre), `USER`, `STAT` (estado), `NI` (prioridad) y `CMD` (comando).
- Anotar el **PID** del que está bloqueado.

**2 · Leer el estado (columna STAT)**

- `R` → ejecutándose.  `S` → esperando (dormido).  `T` → detenido (`Ctrl+Z`).  `Z` → zombi.
- Si lleva `+` (por ejemplo `S+`), está en **primer plano**. Sin `+`, en **segundo plano**.

**3 · Ver quién es su padre**

```
ps -o pid,cmd -p PPID
pstree -p -s PID
```
- Normalmente el padre es `bash`: la terminal desde la que se lanzó.
- `pstree -p -s PID` enseña la cadena de padres hasta `systemd`.

**4 · Comprobar plano desde su terminal**

```
jobs -l
```
- `-l` → añade el PID de cada trabajo.
- Solo funciona en la misma terminal donde se lanzó. Enseña número de trabajo, PID y estado (`Running`, `Stopped`).

**5 · Finalizarlo correctamente**

```
kill PID
```
- Cierre normal: le pide que termine.
- Si está detenido (`T`), usar `kill %1` desde su terminal, que lo reanuda y lo cierra.

**6 · Comprobar que ha desaparecido**

```
ps -p PID
```
- Si solo sale la cabecera (`PID TTY TIME CMD`), ya no existe.

**7 · Si no hubiera respondido**

```
kill -9 PID
```
- Cierre forzado: solo como último recurso, puede dejar datos a medias.
- Con `sudo` delante si el proceso es de otro usuario.

**Informe**

- **Proceso:** nombre y PID
- **Usuario:** columna USER
- **Padre:** PPID y qué es (normalmente bash)
- **Estado y plano:** STAT (con + = primer plano)
- **Prioridad:** NI (0 = normal)
- **Actuación:** `kill PID` y comprobación con `ps -p PID`
- **Si no responde:** `kill -9 PID`, explicando por qué es el último recurso

**Más detalle**

- → [4.5 Trabajos en segundo plano](#e4_5)

<a id="e2_2"></a>

### 2.2 Investigar un proceso desconocido

**Situación:** En la lista de procesos aparece uno cuya función se desconoce. Averiguar qué es, quién lo ejecuta, de dónde viene, si pertenece a un servicio o a un paquete, y valorar si debe detenerse.

**1 · Elegirlo de la lista**

```
ps aux
ps aux --sort=-%cpu | head
```
- `a` → procesos de todos los usuarios. `u` → columnas de usuario, %CPU y %MEM. `x` → incluye los que no tienen terminal.
- `--sort=-%cpu` → ordena de más a menos CPU. `| head` → solo las 10 primeras líneas.
- Elegir uno cuyo nombre no se reconozca y anotar su **PID** (2.ª columna).
- **Comprobación:** la línea del proceso elegido.

**2 · Datos básicos**

```
ps -fp PID
```
- `-f` → formato completo (incluye PPID y el comando entero). `-p` → solo ese PID.
- `UID` → usuario. `PPID` → padre. `TTY` `?` → sin terminal. `CMD` → comando completo (si es un script, aquí sale su ruta).
- **Comprobación:** salida completa.

**3 · Padre y posición en el árbol**

```
ps -fp PPID
pstree -p -s PID
```
- `pstree`: `-p` muestra los PID; `-s` muestra la cadena de padres hasta `systemd`.
- `PPID` 1 → lo lanzó `systemd` (servicio) o lo adoptó porque su padre original terminó.

**4 · Estado y memoria**

```
cat /proc/PID/status
```
- `/proc/PID/` es una carpeta virtual con la información del proceso.
- Líneas útiles: `Name`, `State` (`R` ejecutándose, `S` dormido, `T` detenido, `Z` zombi), `PPid`, `Uid` y `VmRSS` (memoria).

**5 · Ejecutable y directorio de trabajo**

```
sudo readlink -f /proc/PID/exe
sudo readlink -f /proc/PID/cwd
```
- `exe` y `cwd` son enlaces (accesos directos). `readlink` dice a dónde apuntan; `-f` da la ruta completa.
- `exe` → programa real. Si sale `/usr/bin/bash` o `python3`, es un script: su ruta aparece en el `CMD` del paso 2.
- `cwd` → carpeta desde la que trabaja.
- **Comprobación:** las dos rutas.

**6 · ¿Servicio o sesión?**

```
systemctl status PID
```
- Primera línea con `algo.service` → pertenece a ese servicio. Con `session-N.scope` → se lanzó a mano desde una sesión de usuario.
- **Comprobación:** primera línea.

**7 · ¿Usa la red?**

```
sudo ss -tulpn | grep "pid=PID,"
```
- `-t` TCP, `-u` UDP, `-l` solo los que escuchan, `-p` qué proceso, `-n` números en vez de nombres.
- Sin resultado → no escucha en ningún puerto.

**8 · ¿De qué paquete viene?**

```
dpkg -S /ruta/del/ejecutable
```
- `-S` → busca qué paquete instaló ese fichero. Usar la ruta del paso 5.
- Con un paquete → software instalado con `apt`. `no path found` → fichero propio o instalado a mano.
- **Comprobación:** resultado.

**9 · Valorar y decidir**

- Señales para desconfiar: se ejecuta desde `/tmp` o una carpeta personal, no viene de ningún paquete, usa la red sin motivo o corre como `root` sin motivo.
- Un solo dato no basta: se decide con todos juntos. **No finalizar un proceso sin saber qué hace.**
- Si hay que pararlo: proceso suelto → `kill PID` (`kill -9 PID` solo si no responde). Proceso de un servicio → `sudo systemctl stop servicio`, porque `systemd` podría volver a lanzarlo.

**Informe**

- **Proceso investigado:** nombre y PID
- **Función probable:** qué parece hacer
- **Quién lo ejecuta:** UID
- **De dónde procede:** ejecutable, comando, directorio y padre
- **Servicio relacionado:** el de `systemctl status PID`, o ninguno
- **Actividad de red:** puertos de `ss`, o ninguna
- **Paquete relacionado:** el de `dpkg -S`, o ninguno
- **Valoración:** ¿parece normal?
- **Decisión:** ¿se detendría? cómo y por qué

**Conceptos clave**

- **/proc:** Sistema de ficheros virtual: el núcleo muestra ahí los datos de los procesos. No ocupa disco.
- **/proc/2450:** Carpeta con la información del proceso de PID 2450.
- **status:** Nombre, estado, PID, PPID, usuario (Uid) y memoria.
- **cmdline:** Comando completo con el que se lanzó.
- **exe:** Enlace al ejecutable real.
- **cwd:** Carpeta de trabajo del proceso.
- **Al terminar el proceso:** Su carpeta `/proc/PID` desaparece.
- **Utilidad de /proc:** Investigar cualquier proceso sin herramientas extra.
- **PID y PPID:** PID: número único del proceso. PPID: PID de su padre, el que lo lanzó.
- **Padre e hijo:** El padre lanza al hijo. Si el padre muere, el hijo pasa a colgar de `systemd` (PID 1).
- **kill y kill -9:** `kill`: pide al proceso que termine ordenadamente. `kill -9`: lo fuerza; puede dejar datos a medias.
- **Qué justifica investigar más:** Función desconocida, ubicación inesperada, sin paquete, uso de red o permisos elevados sin motivo.

<a id="e2_3"></a>

### 2.3 Provocar y resolver una incidencia en un servicio

**Situación:** Provocar un fallo controlado en un servicio, diagnosticarlo con los registros, corregirlo y demostrar que queda resuelto.

**1 · Partir de un servicio que funciona**

```
systemctl status servicio
```
- Mejor uno propio (apartados 4.2 a 4.4), por ejemplo `aso-practica.service`, para no romper nada del sistema.
- Debe estar `active (running)`.
- **Comprobación:** estado inicial.

**2 · Provocar el fallo, forma A: ruta inexistente**

```
sudo nano /etc/systemd/system/servicio.service
sudo systemctl daemon-reload
sudo systemctl restart servicio
```
- En `ExecStart=`, poner una ruta que no exista, por ejemplo `/usr/local/bin/no-existe.sh`. Guardar con `Ctrl+O` y Enter; salir con `Ctrl+X`.
- `daemon-reload` → obliga a `systemd` a releer el fichero editado.

**3 · Provocar el fallo, forma B: sin permiso de ejecución**

```
sudo chmod -x /ruta/del/script
sudo systemctl restart servicio
```
- `-x` → quita el permiso de ejecución a todos.
- No hace falta `daemon-reload`: el `.service` no ha cambiado.

**4 · Detectar**

```
systemctl status servicio
```
- `Active: failed` → intentó arrancar y falló. Las últimas líneas suelen dar ya la pista.
- **Comprobación:** estado tras el fallo.

**5 · Leer el registro**

```
journalctl -u servicio -n 20 --no-pager
```
- `-u` → solo ese servicio. `-n 20` → últimas 20 líneas. `--no-pager` → todo de golpe.
- Buscar líneas con `Failed`, `status=` o `Result:`.
- **Comprobación:** la línea del error.

**6 · Interpretar el mensaje**

- `status=203/EXEC` → `systemd` no ha podido ejecutar el programa de `ExecStart`: la ruta no existe (forma A) o no tiene `x` (forma B).
- `start-limit-hit` o `Start request repeated too quickly` → falló varias veces seguidas y `systemd` dejó de reintentar.
- `exited, status=1` → el programa arrancó pero terminó con error. Ejecutarlo a mano para ver por qué.
- `Unit file changed on disk` → falta `daemon-reload`.

**7 · Confirmar la causa**

```
systemctl cat servicio
ls -l /ruta/de/ExecStart
pgrep -a nombre
```
- `systemctl cat` → enseña el `.service`; mirar la línea `ExecStart=`.
- `ls -l`: `No such file or directory` → ruta mala. Sin ninguna `x` (`-rw-r--r--`) → falta permiso de ejecución.
- `pgrep -a` → busca el proceso por nombre; `-a` muestra el comando completo. Sin resultado → no se está ejecutando.

**8 · Corregir**

```
sudo nano /etc/systemd/system/servicio.service
sudo systemctl daemon-reload
sudo chmod +x /ruta/del/script
```
- Forma A: devolver `ExecStart` a la ruta correcta y hacer `daemon-reload`.
- Forma B: `chmod +x` (`+x` → añade permiso de ejecución). Sin `daemon-reload`.
- Si apareció `start-limit-hit`: `sudo systemctl reset-failed servicio` (borra el bloqueo).

**9 · Arrancar y comprobar**

```
sudo systemctl restart servicio
systemctl status servicio
```
- Debe quedar `active (running)` con un PID nuevo.
- **Comprobación:** estado final.

**10 · Demostrar con el registro**

```
journalctl -u servicio -n 30 --no-pager
```
- Se ven, en orden: el fallo, los intentos y la línea `Started` sin errores. Eso demuestra que la incidencia está resuelta.
- **Comprobación:** las líneas del fallo y del arranque correcto.

**Informe**

- **Servicio:** nombre
- **Fallo provocado:** qué se cambió
- **Estado tras el fallo:** lo que dice `status`
- **Mensaje clave:** la línea del registro que delata la causa
- **Causa real:** ruta inexistente o falta de `x`
- **Corrección:** qué se hizo y con qué comandos
- **Estado final:** `active (running)` y PID
- **Demostración:** líneas del registro antes y después

**Más detalle**

- → [4.1 Mensajes de error de systemd](#e4_1)
- → [4.2 Crear un servicio propio, paso 1: el script](#e4_2)

**Conceptos clave**

- **status y journalctl:** `status`: cómo está el servicio **ahora**. `journalctl`: **historial** de lo que ha pasado.
- **Demostrar con los registros que se resolvió:** Se ven el fallo, los intentos y después la línea `Started` sin errores.
- **ExecStart:** Ruta del programa que lanza el servicio. Debe existir y tener permiso `x`.
- **daemon-reload:** Hace que `systemd` relea los `.service` después de crearlos o editarlos.
- **203/EXEC:** `systemd` no pudo ejecutar el programa: ruta inexistente o sin permiso de ejecución.
- **Restart=on-failure:** Si el programa falla, `systemd` lo vuelve a lanzar.
- **reset-failed:** Borra el bloqueo que deja `systemd` tras varios fallos seguidos.

<a id="e2_4"></a>

### 2.4 Auditar un servicio

**Situación:** Elegir un servicio del sistema y documentarlo: unidad, proceso, PID, usuario, ejecutable, paquete, arranque automático, dependencias y registros. Pararlo y arrancarlo comprobando qué cambia.

**1 · Elegir un servicio**

```
systemctl list-units --type=service --state=running
```
- `--type=service` → solo servicios. `--state=running` → solo los que están en marcha.
- Elegir uno inofensivo, por ejemplo `cron`. **No** usar `ssh` si se trabaja conectado por SSH: se corta la conexión.

**2 · Estado**

```
systemctl status servicio
```
- `Loaded:` → ruta del `.service` y si está `enabled` (arranca solo). `Active:` → estado. `Main PID:` → PID principal. Al final, los últimos mensajes.
- `q` para salir, o añadir `--no-pager`.
- **Comprobación:** salida completa.

**3 · La unidad**

```
systemctl cat servicio
```
- Muestra el fichero `.service`. Líneas clave: `Description=`, `ExecStart=` (programa que lanza), `Restart=`, `After=` o `Requires=` (dependencias) y `WantedBy=` (con qué objetivo arranca).
- **Comprobación:** el fichero.

**4 · Proceso, PID y usuario**

```
ps -fp PID
```
- Con el PID de `Main PID`. `-f` → formato completo. `-p` → solo ese PID.
- `UID` → usuario que lo ejecuta (muchos servicios usan `root`).
- Si tiene varios procesos, `systemctl status` los muestra en el árbol `CGroup:`.

**5 · Ejecutable y paquete de origen**

```
sudo readlink -f /proc/PID/exe
dpkg -S /ruta/del/ejecutable
```
- `readlink -f` → ruta real del programa; debe coincidir con `ExecStart`.
- `dpkg -S` → paquete que instaló ese fichero.
- **Comprobación:** las dos salidas.

**6 · Arranque automático**

```
systemctl is-enabled servicio
```
- `enabled` → arranca con el sistema. `disabled` → no. `static` → no se activa solo; lo arranca otra unidad.

**7 · Dependencias**

```
systemctl list-dependencies servicio
```
- Árbol de unidades que necesita. Basta con señalar una.
- Al revés (quién depende de él): añadir `--reverse`.

**8 · Registros**

```
journalctl -u servicio -n 20 --no-pager
```
- `-u` → solo ese servicio. `-n 20` → últimas 20 líneas. `--no-pager` → todo de golpe.
- Solo desde el último arranque del equipo: `-b` en lugar de `-n 20`.
- **Comprobación:** últimas líneas.

**9 · Detener y comprobar**

```
sudo systemctl stop servicio
systemctl is-active servicio
ps -p PID
```
- `is-active` → debe responder `inactive`.
- `ps -p PID` solo muestra la cabecera → el proceso ha desaparecido.
- **Comprobación:** las tres salidas.

**10 · Arrancar y comparar el PID**

```
sudo systemctl start servicio
systemctl status servicio
```
- Vuelve `active (running)` con **otro PID**: es un proceso nuevo, el anterior terminó.
- En `journalctl -u servicio -n 10 --no-pager` se ven el `Stopped` y el `Started`.
- **Comprobación:** estado final y PID nuevo.

**Informe**

- **Servicio analizado:** nombre y descripción
- **Fichero de unidad:** ruta (línea `Loaded:`)
- **Proceso asociado:** nombre, PID y ejecutable
- **Usuario:** UID
- **Paquete:** el de `dpkg -S`
- **Arranque automático:** enabled, disabled o static
- **Dependencia:** al menos una
- **Estado inicial:** Active
- **Actuación:** stop → comprobación → start
- **Estado final:** Active y PID nuevo (y por qué cambia)

**Más detalle**

- → [4.2 Crear un servicio propio, paso 1: el script](#e4_2)

**Conceptos clave**

- **Proceso, demonio y servicio:** Proceso: programa en ejecución. Demonio: proceso en segundo plano sin terminal. Servicio: demonio gestionado por `systemd`.
- **systemd:** Primer proceso del sistema (PID 1): arranca y vigila los servicios.
- **systemctl:** Comando para gestionar servicios: ver estado, arrancar, parar, habilitar.
- **systemctl status:** Estado, si arranca solo, PID principal, memoria y últimos mensajes.
- **start y enable:** `start`: arranca **ahora**. `enable`: arrancará **en cada inicio** (no lo arranca ahora).
- **enabled:** Arrancará con el sistema. **No** significa que esté funcionando ahora.
- **restart y reload:** `restart`: lo para y lo arranca (PID nuevo). `reload`: relee la configuración sin pararlo.
- **Al detener un servicio:** Su proceso termina y desaparece.
- **El PID cambia al reiniciar:** Porque se crea un proceso nuevo.
- **Unidad .service:** Fichero que dice a `systemd` qué programa lanzar, cómo y cuándo.
- **Dependencias:** Fijan qué debe arrancar antes (`After=`) y con qué objetivo arranca (`WantedBy=`). Evitan arrancar sin lo necesario.
- **Script directo o con systemd:** Directo: depende de la terminal y nadie lo vigila. Con `systemd`: arranca solo, se reinicia y deja registro.
- **.service → systemd → proceso → PID:** `systemd` lee el fichero, lanza el programa de `ExecStart` y ese proceso recibe un PID.

<a id="e2_5"></a>

### 2.5 Servidor lento

**Situación:** «El servidor va muy lento.» Primera comprobación: CPU, RAM, disco, procesos que más consumen y servicios. Decir qué se busca en cada uno.

**1 · CPU**

```
top
```
- Línea `%Cpu(s)`, valor `id`: es la CPU **libre** (al revés que en Windows). `id` cerca de 0 → CPU saturada.
- Columna `%CPU` de cada proceso: su uso real. 100 % = un núcleo entero, así que puede pasar de 100.
- Dentro de `top`: `P` ordena por CPU, `M` por memoria, `q` sale.

**2 · Procesos que más consumen**

```
ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu | head
ps -eo pid,user,%cpu,%mem,cmd --sort=-%mem | head
```
- `-e` → todos los procesos. `-o` → columnas elegidas. `--sort=-%cpu` → de más a menos CPU. `| head` → 10 primeras líneas.
- El primero de la lista es el que más gasta. Anotar `PID`, `USER` y `CMD`.

**3 · RAM**

```
free -h
```
- `-h` → en GB y MB, legible.
- Mirar la columna `available` (no `free`). Si es muy baja y `Swap` está muy usada, falta memoria.

**4 · Disco**

```
df -h
du -sh /ruta/*
```
- En `df`, la línea `/`: un `Use%` cerca del 100 % ralentiza.
- `du -sh` dice qué carpetas ocupan más.

**5 · Servicios**

```
systemctl --type=service --state=running
systemctl --failed
```
- Buscar servicios fallidos o que no deberían estar en marcha.

**6 · Decidir**

```
systemctl status PID
```
- Si el proceso es de un servicio, mirar su estado y su registro antes de tocar nada.
- Bajar su prioridad sin finalizarlo: `sudo renice -n 19 -p PID` (`-n 19` → prioridad mínima; `-p` → el PID).
- Pararlo solo si se sabe qué es. Proceso suelto → `kill PID`. Proceso de un servicio → `sudo systemctl stop servicio`.

**Informe**

- **En cada punto:** comando + qué se busca + qué se ha encontrado
- **Proceso señalado:** nombre, PID y usuario
- **Decisión:** finalizar, bajar prioridad o dejarlo, con el motivo

<a id="e2_6"></a>

### 2.6 Windows: aplicación bloqueada y servicio parado

**Situación:** En Windows, una aplicación se ha bloqueado y un servicio parece detenido. Cerrar solo esa aplicación y arrancar el servicio.

**1 · Abrir PowerShell como administrador**

- Sin eso, parar o arrancar servicios da «acceso denegado».

**2 · Localizar la aplicación y su PID**

```
Get-Process -Name nombre
```
- Nombre sin `.exe` (por ejemplo `notepad`). Columna `Id` = PID.

**3 · Ver su consumo**

```
Get-Process -Id PID
```
- `CPU(s)` → segundos de CPU usados. `WS(K)` → memoria.

**4 · Finalizar solo ese proceso**

```
Stop-Process -Id PID
```
- Por PID, no por nombre: `-Name` cerraría todas las ventanas con ese nombre. Si no se cierra, añadir `-Force`.

**5 · Comprobar que ha desaparecido**

```
Get-Process -Id PID
```
- Si da error «no se encuentra ningún proceso», ha desaparecido.

**6 · Estado del servicio**

```
Get-Service nombre
```
- `Status`: `Running` (en marcha) o `Stopped` (parado). Ejemplo: `Get-Service Spooler`.

**7 · Iniciarlo y comprobar**

```
Start-Service nombre
Get-Service nombre
```
- Debe pasar a `Running`.

**Informe**

- **Proceso investigado:** nombre
- **PID:** Id
- **Actuación realizada:** `Stop-Process -Id PID` y comprobación
- **Servicio investigado:** nombre
- **Estado inicial:** Stopped
- **Estado final:** Running

<a id="e2_7"></a>

### 2.7 Instalar o eliminar un paquete

**Situación:** Instalar una herramienta nueva, o quitar una aplicación que ya no se usa, comprobando procesos, dependencias y espacio.

**1 · ¿Está instalado?**

```
dpkg -l | grep paquete
which programa
```
- `dpkg -l` → lista los paquetes instalados. `| grep paquete` → deja solo las líneas con ese nombre.
- `ii` al principio → instalado. `which` da la ruta del ejecutable (si no sale nada, no está).

**2 · Instalar**

```
sudo apt update
sudo apt install paquete
```
- `update` → actualiza la lista de paquetes (no instala nada).
- Antes de confirmar, leer «se instalarán los siguientes paquetes adicionales»: eso son las dependencias que hay que documentar.

**3 · Comprobar que funciona y si deja procesos**

```
programa --version
pgrep -a programa
```

**4 · Eliminar**

```
pgrep -a programa
sudo apt remove paquete
```
- Si tiene procesos en marcha, cerrarlos antes.
- `remove` conserva la configuración. `sudo apt purge paquete` la borra también.

**5 · Comprobar que ha desaparecido**

```
which programa
dpkg -l | grep paquete
```
- `which` sin resultado. En `dpkg -l`, nada o `rc` (borrado, pero queda su configuración).

**6 · Dependencias sobrantes y espacio**

```
sudo apt autoremove
df -h
```
- Leer la lista antes de confirmar con `s`/`y`: no confirmar a ciegas.

**Informe**

- **Por cada paso:** el comando y lo que se ha visto
- **Paquetes adicionales:** los que listó apt al instalar o al hacer autoremove

<a id="e2_8"></a>

### 2.8 SSH: conectar y diagnosticar

**Situación:** «No puedo conectarme al servidor por SSH», o conectarse a un servidor y hacer comprobaciones en remoto.

**1 · Orden si no conecta: IP del servidor**

```
ip a
```
- En el servidor. También vale `hostname -I`.

**2 · Conectividad**

```
ping IP
```
- Desde el cliente. `Ctrl+C` para parar. Si no responde, el problema es de red.

**3 · Estado del servicio SSH**

```
systemctl status ssh
systemctl status ssh.socket
```
- En Ubuntu reciente, `ssh.service` puede salir `inactive` y estar todo bien: escucha `ssh.socket` y arranca el servicio al llegar una conexión.

**4 · Proceso del servicio**

```
pgrep -a sshd
```

**5 · Puerto en escucha**

```
sudo ss -tlnp | grep :22
```
- `-t` TCP, `-l` solo los que escuchan, `-n` números en vez de nombres, `-p` qué proceso. `grep :22` → solo el puerto 22.
- Debe salir una línea `LISTEN` en el puerto 22.
- 22 es el puerto por defecto; a menudo se cambia para esquivar escaneos automáticos (línea `Port` de `/etc/ssh/sshd_config`). Para ver el real: `sudo ss -tlnp | grep ssh`. Con otro puerto: `ssh -p puerto usuario@IP`.

**6 · Usuario**

```
ssh usuario@IP
id usuario
```
- El usuario debe existir **en el servidor** (`id usuario`, ejecutado en el servidor).

**7 · Comprobaciones en remoto**

```
hostname
whoami
free -h
df -h
ps aux | head
exit
```
- Nombre del servidor, usuario actual, memoria, disco y procesos. `exit` cierra la sesión remota.

**8 · Aviso**

- **No** parar `ssh` estando conectado por SSH: se corta la propia conexión.

**Informe**

- **Por cada comprobación:** comando, dónde se ejecuta (cliente o servidor) y resultado

<a id="e2_9"></a>

### 2.9 Carpetas, ficheros y datos del equipo

**Situación:** Crear una estructura de carpetas, crear, copiar, renombrar y borrar ficheros, y sacar datos básicos del equipo.

**1 · Estructura de carpetas**

```
mkdir -p empresa/{usuarios,copias,temporal}
mkdir -p empresa/departamentos/{administracion,ventas,informatica}
```
- `-p` crea también las carpetas de encima. Las llaves crean varias de golpe (sin espacios dentro).

**2 · Crear fichero, copiar y renombrar**

```
touch empresa/usuarios.txt
cp empresa/usuarios.txt empresa/copias/
mv empresa/copias/usuarios.txt empresa/copias/usuarios_backup.txt
```
- `mv` sirve tanto para mover como para renombrar.

**3 · Escribir líneas en un fichero**

```
echo "ana:administracion" > usuarios.txt
echo "juan:ventas" >> usuarios.txt
```
- `>` **borra** lo que había y escribe. `>>` añade al final.

**4 · Borrar una carpeta**

```
rm -r empresa/temporal
```
- Si está vacía, también `rmdir empresa/temporal`.

**5 · Comprobar**

```
ls -R empresa
```
- Lista todo lo que hay dentro. Si está instalado, `tree empresa` lo dibuja en árbol.

**6 · Datos del equipo**

```
hostname
whoami
cat /etc/os-release
pwd
hostname -I
ip route show default
```
- Nombre, usuario, versión de Linux, carpeta actual, IP y puerta de enlace (la IP tras `via`).
- Espacio en disco: `df -h`. Memoria: `free -h`.

**Informe**

- **Por cada paso:** el comando y la comprobación (`ls -R`, `cat`)

<a id="e2_10"></a>

### 2.10 Buscar un fichero y filtrar su contenido

**Situación:** Localizar un fichero (por ejemplo `usuarios_empresa.txt`), quedarse solo con las líneas de un departamento y guardarlas en otro fichero.

**1 · Localizar el fichero**

```
find ~ -name "usuarios_empresa.txt"
```
- Primero dónde buscar (`~`, `/`, `~/ASO`), luego `-name`.
- En todo el sistema: `find / -name "nombre" 2>/dev/null` (oculta los errores de permiso).
- Comodines: `-name "*.txt"` (terminan en .txt), `-name "*usuario*"` (contienen usuario). `-iname` ignora mayúsculas.

**2 · Filtrar líneas**

```
grep ventas /ruta/usuarios_empresa.txt
```
- `-i` ignora mayúsculas. `-c` cuenta las líneas. `-n` enseña el número de línea.

**3 · Guardar el resultado**

```
grep ventas /ruta/usuarios_empresa.txt > usuarios_ventas.txt
```
- `>` crea o sobrescribe el fichero. `>>` añadiría al final.

**4 · Comprobar**

```
cat usuarios_ventas.txt
wc -l usuarios_ventas.txt
```
- `wc -l` cuenta líneas: debe coincidir con `grep -c ventas` sobre el original.

**5 · Leer ficheros**

```
less archivo
head -n 2 archivo
tail -n 2 archivo
```
- `cat` lo vuelca entero. `less` permite moverse y buscar con `/` (`q` sale). `head` → principio. `tail` → final.

**6 · Tuberías**

```
cat usuarios.txt | grep ventas
history | grep cd
```
- `|` pasa la salida del comando de la izquierda como entrada del de la derecha.

**Informe**

- **Por cada paso:** el comando y su salida
- **find frente a grep:** `find` busca **ficheros** por nombre; `grep` busca **texto dentro** de ficheros

<a id="e2_11"></a>

### 2.11 Leer y cambiar permisos

**Situación:** Interpretar permisos (`rwxr-xr--`, `755`…), cambiarlos, cambiar propietario y explicar por qué no usar `chmod 777`.

**1 · Leer los permisos**

```
ls -l archivo
ls -ld carpeta
```
- Ejemplo: `-rw-r--r-- 1 jordi jordi … informe.txt`.
- 1.er carácter: `-` fichero, `d` directorio. Después, tres grupos de tres: **propietario**, **grupo**, **otros**. Luego, nombre del propietario y del grupo.
- Para ver los permisos de la propia carpeta (y no de lo que contiene): `ls -ld`.

**2 · Qué significa cada letra**

- En fichero: `r` leer, `w` modificar, `x` ejecutar.
- En carpeta: `r` listar, `w` crear o borrar dentro, `x` entrar. **La `x` no significa lo mismo.**

**3 · Números**

- `r`=4, `w`=2, `x`=1, y se suman: 7 = `rwx`, 6 = `rw-`, 5 = `r-x`, 4 = `r--`, 0 = `---`.
- `755` = `rwxr-xr-x`. `644` = `rw-r--r--`. `640` = `rw-r-----`. `700` = `rwx------`. `750` = `rwxr-x---`. `770` = `rwxrwx---`.

**4 · Cambiar permisos**

```
chmod 640 informe.txt
chmod u+x script.sh
chmod o-rwx carpeta
```
- Con letras: `u` propietario, `g` grupo, `o` otros; `+` añade, `-` quita.
- `chmod` cambia los permisos del **fichero**, nunca los de un usuario. Lo que puede hacer un usuario depende de en qué bloque cae: propietario (`chown`), grupo (`usermod -aG`) u otros.

**5 · Cambiar propietario o grupo**

```
sudo chown usuario1 informe.txt
sudo chgrp ventas informe.txt
sudo chown usuario1:ventas informe.txt
```
- Necesita `sudo`: un usuario normal no puede regalar sus ficheros.

**6 · Por qué no chmod 777**

- Da todos los permisos a **todos** los usuarios. Arregla el síntoma pero abre la puerta a cualquiera.
- Antes: mirar propietario y grupo (`ls -l`), los grupos del usuario (`id usuario`) y dar **solo** lo necesario.

**Informe**

- **Interpretar:** propietario / grupo / otros, cada uno con lo que puede hacer
- **Cambiar:** comando + `ls -l` que lo demuestra

<a id="e2_12"></a>

### 2.12 Carpeta para un departamento

**Situación:** Crear grupos y usuarios, y carpetas a las que solo puedan entrar, leer y modificar los miembros de su grupo (por ejemplo `ventas` y `administracion`).

**1 · Crear los grupos**

```
sudo groupadd ventas
sudo groupadd administracion
getent group ventas
```

**2 · Crear los usuarios**

```
sudo adduser usuario1
sudo adduser usuario2
```

**3 · Meter cada usuario en su grupo**

```
sudo usermod -aG ventas usuario1
sudo usermod -aG administracion usuario2
id usuario1
```
- `-aG` **añade**. Sin la `a`, el usuario pierde sus otros grupos.

**4 · Crear las carpetas**

```
mkdir -p ~/ASO/empresa/{ventas,administracion}
```

**5 · Asignar el grupo a cada carpeta**

```
sudo chgrp ventas ~/ASO/empresa/ventas
sudo chgrp administracion ~/ASO/empresa/administracion
```

**6 · Permisos: grupo sí, resto no**

```
chmod 770 ~/ASO/empresa/ventas
chmod 770 ~/ASO/empresa/administracion
```
- `770` → propietario y grupo: entrar, leer y modificar. Otros: nada.

**7 · Comprobar**

```
ls -l ~/ASO/empresa
```
- Debe salir `drwxrwx--- … jordi ventas … ventas` (y lo mismo con administracion).

**8 · Detalle importante**

- Para llegar a la carpeta, `usuario1` también necesita `x` en las de encima (`/home/jordi`, `~/ASO`). En Ubuntu reciente `/home/jordi` es `750`, así que habría que tenerlo en cuenta.
- Los grupos nuevos se aplican al volver a iniciar sesión.

**Informe**

- **Usuarios y grupos:** `id usuario1` → aparece ventas
- **Carpetas:** propietario, grupo y permisos según `ls -l`
- **Por qué 770:** mínimo necesario: el grupo trabaja, el resto no entra

<a id="s3"></a>

## 3. Conceptos generales

<a id="e3_1"></a>

### 3.1 Procesos

- **Programa y proceso:** Programa = fichero en disco. Proceso = programa en ejecución.
- **Primer y segundo plano:** Primer plano: ocupa la terminal. Segundo plano (`&`): la terminal queda libre.
- **jobs y ps:** `jobs`: trabajos de **esta** terminal. `ps`: procesos del sistema.
- **fg y bg:** `fg`: trae un trabajo al frente. `bg`: lo reanuda en segundo plano.
- **kill:** Envía una señal a un proceso. Por defecto le pide que termine; `-9` lo fuerza.
- **Valor nice:** Prioridad: de -20 (máxima) a 19 (mínima). 0 es lo normal.
- **ps, top y htop:** `ps`: foto fija. `top`: en tiempo real. `htop`: como `top`, interactivo y con colores.
- **Saber qué usuario ejecuta un proceso:** Indica sus privilegios y ayuda a detectar algo anómalo.
- **Finalizar cualquier proceso:** Un usuario normal, solo los suyos. Con `sudo`, cualquiera, con riesgo de romper el sistema.

<a id="e3_2"></a>

### 3.2 Paquetes

- **Gestor de paquetes:** Herramienta que instala, actualiza y quita software resolviendo dependencias (`apt`).
- **apt update y apt upgrade:** `update`: actualiza la lista de paquetes. `upgrade`: instala las versiones nuevas.
- **remove y purge:** `remove`: quita el programa y deja la configuración. `purge`: quita también la configuración.
- **Dependencia:** Paquete que otro necesita para funcionar.
- **apt autoremove:** Quita dependencias que ya nadie usa.
- **which:** Ruta del ejecutable de un comando.
- **No confirmar a ciegas:** Puede instalar o borrar más paquetes de los esperados.

<a id="e3_3"></a>

### 3.3 Ficheros y búsqueda

- **cat y less:** `cat`: vuelca todo. `less`: permite moverse y buscar.
- **head y tail:** `head`: primeras líneas. `tail`: últimas (útil en logs).
- **> y >>:** `>`: sobrescribe. `>>`: añade al final.
- **Tubería |:** Pasa la salida de un comando como entrada del siguiente.
- **Buscar un fichero o buscar dentro:** `find`: localiza ficheros por nombre. `grep`: busca texto dentro.
- **df y du:** `df`: espacio de cada disco. `du`: lo que ocupa una carpeta.
- **free y df:** `free`: memoria RAM y swap. `df`: espacio en disco.

<a id="e3_4"></a>

### 3.4 Usuarios y permisos

- **Usuario y grupo:** Usuario: cuenta individual con su UID. Grupo: conjunto de usuarios que comparten permisos.
- **id:** UID, grupo principal y todos los grupos del usuario.
- **whoami e id:** `whoami`: solo el nombre. `id`: además UID, GID y grupos.
- **chmod y chown:** `chmod`: cambia permisos. `chown`: cambia propietario y/o grupo.
- **Tres bloques de ls -l:** Propietario, grupo y otros, en ese orden.
- **No dar permisos de más:** Seguridad: cada permiso extra es una puerta para un error o un ataque.

<a id="e3_5"></a>

### 3.5 Red, SSH y Windows

- **Dirección IP:** Número que identifica una interfaz en la red.
- **DNS:** Traduce nombres de dominio a direcciones IP.
- **SSH:** Protocolo cifrado para administrar equipos en remoto (puerto 22).
- **Equivalencias en Windows:** `ps` → `Get-Process`. `kill` → `Stop-Process`. `systemctl status` → `Get-Service`. `start`/`stop` → `Start-Service`/`Stop-Service`.

<a id="s4"></a>

## 4. Referencia rápida

<a id="e4_1"></a>

### 4.1 Mensajes de error de systemd

Cada bloque es un mensaje típico del log, lo que significa y qué hacer.

```
ls -l /ruta/del/script
```
- El comando de arriba es el del primer bloque: enseña los permisos de un fichero. Si la primera columna es `-rw-r--r--` (ninguna `x`), el fichero **no se puede ejecutar**; con `-rwxr-xr-x` sí.

- **Se ve:** `status=203/EXEC`
- **Significa:** `systemd` no ha podido ejecutar el programa que indica `ExecStart`: la ruta está mal escrita o el fichero no tiene permiso de ejecución.
- **Hacer:** Mirar los permisos con `ls -l /ruta/del/script`. Si falta la `x`: `sudo chmod +x /ruta/del/script`. Si la ruta está mal: `sudo nano /etc/systemd/system/servicio.service` y corregir `ExecStart`.

- **Se ve:** `Start request repeated too quickly` o `start-limit-hit`
- **Significa:** El servicio ha fallado varias veces seguidas y `systemd` ha dejado de reintentar.
- **Hacer:** Corregir primero la causa (por ejemplo, el bloque anterior). Después limpiar el bloqueo: `sudo systemctl reset-failed servicio` y arrancar: `sudo systemctl start servicio`.

- **Se ve:** `exited, status=1` (u otro número distinto de 0)
- **Significa:** El servicio arrancó, pero el programa o script ha terminado con error.
- **Hacer:** Ejecutarlo a mano para ver el error: `sudo /ruta/del/script`.

- **Se ve:** `Unit servicio.service not found` o `could not be found`
- **Significa:** `systemd` no conoce ese servicio: nombre mal escrito, o fichero `.service` recién creado sin recargar.
- **Hacer:** Comprobar el nombre. Si el fichero es nuevo: `sudo systemctl daemon-reload`.

- **Se ve:** `Unit file changed on disk`
- **Significa:** Se ha editado el `.service` y `systemd` todavía usa la versión antigua.
- **Hacer:** `sudo systemctl daemon-reload` y volver a arrancar el servicio.

- **Se ve:** `Permission denied` al editar o ejecutar
- **Significa:** Falta permiso: casi siempre falta `sudo`.
- **Hacer:** Repetir el comando con `sudo` delante.

<a id="e4_2"></a>

### 4.2 Crear un servicio propio, paso 1: el script

Un servicio necesita un programa que lance. Aquí, un script que escribe la hora cada 10 segundos. El nombre `mi-servicio` es libre.

```
#!/bin/bash
while true
do
echo "$(date) - Servicio funcionando"
sleep 10
done
```
- Crear el fichero con `sudo nano /usr/local/bin/mi-servicio.sh` y pegar el contenido de arriba.
- Darle permiso de ejecución: `sudo chmod +x /usr/local/bin/mi-servicio.sh`. **Sin esto, el servicio dará `203/EXEC`.**

<a id="e4_3"></a>

### 4.3 Crear un servicio propio, paso 2: la unidad

Es el fichero que le dice a `systemd` qué lanzar y cuándo.

```
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
- Crear el fichero con `sudo nano /etc/systemd/system/mi-servicio.service` y pegar el contenido de arriba.
- `ExecStart` debe llevar la ruta completa del script del paso 1.

<a id="e4_4"></a>

### 4.4 Crear un servicio propio, paso 3: activarlo

Cargar el fichero nuevo, arrancarlo y comprobar que escribe en el log.

```
sudo systemctl daemon-reload
sudo systemctl enable --now mi-servicio.service
systemctl status mi-servicio.service
journalctl -u mi-servicio.service -n 10 --no-pager
```
- `daemon-reload` → obligatorio tras crear o editar un `.service`.
- `enable --now` → lo activa para el arranque y lo arranca ahora.
- `status` debe decir `active (running)` y `journalctl` debe mostrar una línea nueva cada 10 segundos.

<a id="e4_5"></a>

### 4.5 Trabajos en segundo plano

Qué hacer cuando un comando se queda ocupando la terminal.

```
comando &
jobs
bg %1
fg %1
kill %1
```
- `Ctrl+C` → cancela el comando que está ocupando la terminal.
- `Ctrl+Z` → lo suspende (queda `Stopped`, no muere).
- `comando &` → lanza el comando en segundo plano, y la terminal queda libre.
- `jobs` → lista los trabajos de esa terminal con su número entre corchetes, como `[1]`.
- `bg %1` → reanuda el trabajo 1 en segundo plano. `fg %1` → lo trae al frente. `kill %1` → lo termina.
- Desde otra terminal: `ps aux | grep nombre` para localizar su PID y `kill PID`.
