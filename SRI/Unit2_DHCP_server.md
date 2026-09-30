# Instalación y configuración de DHCP con Kea

>  Antes de instalar: comprobar que no hay otro servicio DHCP activo en la misma red (evita conflictos de IP).

## 1. Instalar

```bash
sudo apt update
sudo apt install kea
```

Durante la instalación aparece un prompt para configurar la contraseña de `kea-ctrl-agent` (su API REST). Elegir **"configure with a given password"** e introducir una propia.

Una contraseña generada al azar queda guardada en un fichero y hay que consultarla cada vez que se necesita (`sudo cat /etc/kea/kea-api-password`). Poniéndola, se conoce desde el principio y se pueds usar directamente si en el futuro se accede a la API (`kea-shell`, recarga en caliente, herramientas de gestión), sin depender de tener acceso root para consultarla.

## 2. Configurar interfaz y rango
Hacer copia de seguridad del fichero a editar:
```yaml
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak
sudo nano /etc/kea/kea-dhcp4.conf
```
```bash
sudo nano /etc/kea/kea-dhcp4.conf
```
Pueden rellenarse los espacios en blanco del propio archivo, o borrarlo todo e insertar el código propuesto  

```json
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

## 3. Validar sintaxis

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

Detecta errores de JSON (comas de más/menos) sin arrancar el servicio.

## 4. Arrancar y comprobar

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Debe salir `active (running)`. Si falla:

```bash
journalctl -u kea-dhcp4-server -e
```

## 5. Verificar desde un cliente

```bash
ip a
```

Si no recibe IP:

```bash
sudo dhclient -r && sudo dhclient
```
## Configurar para autoarranque
---

| Síntoma | Causa probable |
|---|---|
| El servicio no arranca | Error de sintaxis JSON — revisa con `kea-dhcp4 -t` |
| Arranca pero no reparte IPs | Interfaz equivocada, u otro DHCP compitiendo en la red |
| El cliente no recibe IP | VMs en redes distintas (modo de red de VirtualBox) |
