# Instalación y configuración de DHCP con Kea

#instalación-y-configuración-de-dhcp-con-kea
> Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red (evita conflictos de IP).

## 1. Instalar

#1-instalar

```
sudo apt update
sudo apt install kea
```

Durante la instalación aparece un prompt para configurar la contraseña de `kea-ctrl-agent` (su API REST). Elegir **"configure with a given password"** e introducir una propia.

Una contraseña generada al azar queda guardada en un fichero y hay que consultarla cada vez que se necesita (`sudo cat /etc/kea/kea-api-password`). Poniéndola, se conoce desde el principio y se puede usar directamente si en el futuro se accede a la API (`kea-shell`, recarga en caliente, herramientas de gestión), sin depender de tener acceso root para consultarla.

## 2. Configurar interfaz y rango

#2-configurar-interfaz-y-rango

Hacer copia de seguridad del fichero a editar:

```
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak
```

> **Recomendado:** en vez de editar con `nano` (el copiar y pegar desde otro sitio puede introducir comillas tipográficas o cortar líneas, provocando errores de sintaxis difíciles de ver), sobrescribir el fichero de una vez con:
> ```
> sudo tee /etc/kea/kea-dhcp4.conf > /dev/null << 'EOF'
> { ... contenido del JSON de abajo ... }
> EOF
> ```
> Las comillas simples en `'EOF'` evitan que el terminal interprete el contenido, lo escribe tal cual (End Of File).

Alternativamente, puede editarse a mano:

```
sudo nano /etc/kea/kea-dhcp4.conf
```

Pueden rellenarse los espacios en blanco del propio archivo, o borrarlo todo e insertar el código propuesto:

```
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": ["enp0s3"]
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "lease-database": {
      "type": "memfile",
      "lfc-interval": 3600
    },
    "valid-lifetime": 600,
    "max-valid-lifetime": 7200,
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.1.0/24",
        "pools": [
          { "pool": "192.168.1.100 - 192.168.1.200" }
        ],
        "option-data": [
          { "name": "routers", "data": "192.168.1.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8, 8.8.4.4" }
        ]
      }
    ]
  }
}
```

Sustituir interfaz (`ip a` para verla), red y rango por los propios.

### Cambiar tiempos de concesión (lease time)

[#cambiar-tiempos-de-concesión-lease-time](#cambiar-tiempos-de-concesión-lease-time)

```json
"valid-lifetime": 600,
"max-valid-lifetime": 7200,
```

`valid-lifetime`: duración por defecto de una concesión, en segundos. `max-valid-lifetime`: el máximo permitido, aunque el cliente pida más.

Editar en `/etc/kea/kea-dhcp4.conf`, validar y aplicar sin cortar el servicio:

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl reload kea-dhcp4-server
```

## 3. Validar sintaxis

#3-validar-sintaxis

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

Detecta errores de JSON (comas de más/menos) sin arrancar el servicio.

### Incidencia conocida: "Unable to open file"

Si el comando anterior devuelve `Syntax check failed with: Unable to open file /etc/kea/kea-dhcp4.conf` aunque el fichero exista con permisos correctos (`root:root`, `644`), el bloqueo no es de sintaxis ni de permisos: es el perfil de **AppArmor** del paquete, que impide al proceso `kea-dhcp4` leer el fichero.

Confirmarlo revisando el log del kernel:

```
sudo dmesg | grep -i apparmor | grep -i kea
```

Si aparecen líneas `DENIED` con `capname="dac_override"` o `dac_read_search`, se confirma la causa.

Solución:

```
sudo apt install apparmor-utils
sudo aa-complain /usr/sbin/kea-dhcp4
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

`aa-complain` pone el perfil en modo "registra pero no bloquea", en vez de desactivarlo del todo. Para producción sería preferible ajustar el perfil (`/etc/apparmor.d/usr.sbin.kea-dhcp4`) y volver a modo estricto con `sudo aa-enforce /usr/sbin/kea-dhcp4`; en un entorno de prácticas, dejarlo en `complain` es suficiente.

## 4. Arrancar y comprobar

#4-arrancar-y-comprobar

```
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Debe salir `active (running)`. Si falla:

```
journalctl -u kea-dhcp4-server -e
```

## 5. Verificar desde un cliente

#5-verificar-desde-un-cliente

```
ip a
```

Para renovar la IP en un cliente Ubuntu de escritorio (con NetworkManager):

```
sudo nmcli device disconnect enp0s3
sudo nmcli device connect enp0s3
```

(Sustituir `enp0s3` por la interfaz real del cliente.)

> **Nota:** si un cliente vuelve a solicitar IP y recibe la misma que antes, es el comportamiento normal de DHCP ("sticky lease"): mientras la concesión no haya caducado (`valid-lifetime`) y la MAC coincida, el servidor intenta devolver la misma IP. No es un fallo.

Para ver las IPs concedidas por Kea:

```
sudo cat /var/lib/kea/kea-leases4.csv
```

## Configurar para autoarranque

#configurar-para-autoarranque

```
sudo systemctl enable kea-dhcp4-server
sudo systemctl start kea-dhcp4-server
```

Comprobar que quedó activado:

```
systemctl is-enabled kea-dhcp4-server
```

Debe responder `enabled`. Es probable que ya lo esté por defecto tras la instalación; en ese caso no hace falta tocar nada.

---

| Síntoma                     | Causa probable                                                                                     |
| ---------------------------- | --------------------------------------------------------------------------------------------------- |
| El servicio no arranca      | Error de sintaxis JSON (`kea-dhcp4 -t`), o AppArmor bloqueando el fichero (ver incidencia arriba) |
| Arranca pero no reparte IPs | Interfaz equivocada, u otro DHCP compitiendo en la red                                             |
| El cliente no recibe IP     | VMs en redes distintas (modo de red de VirtualBox)                                                 |


## Consultas habituales de administración

```
sudo systemctl status kea-dhcp4-server
```
Estado del servicio: activo, parado, o con errores recientes.

```
sudo cat /var/lib/kea/kea-leases4.csv
```
Listado de todas las concesiones de IP activas, con su MAC y fecha de caducidad.

```
sudo journalctl -u kea-dhcp4-server -f
```
Sigue el log del servicio en tiempo real; útil para ver en directo cómo llegan las peticiones DHCP de los clientes.

```
sudo journalctl -u kea-dhcp4-server --since "10 min ago"
```
Log de los últimos minutos, sin tener que revisar todo el histórico.

```
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```
Valida la configuración actual sin reiniciar el servicio (repetir tras cualquier cambio manual del fichero).

```
sudo systemctl reload kea-dhcp4-server
```
Aplica cambios de configuración sin cortar las concesiones ya activas (a diferencia de `restart`, que sí interrumpe el servicio un instante).

```
ip -s -4 addr show enp0s3
```
Estadísticas de la interfaz por la que escucha el servidor; útil para confirmar que recibe tráfico.

```
sudo grep <MAC_del_cliente> /var/lib/kea/kea-leases4.csv
```
Busca la concesión de un cliente concreto por su dirección MAC (visible con `ip a` en el propio cliente).
