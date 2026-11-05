# Docker Desktop en una MV de VirtualBox: activar KVM (virtualización anidada)

> **Resumen:** Docker Desktop para Linux necesita KVM. Dentro de una MV de VirtualBox sobre Windows 11, KVM no aparece mientras Windows tenga su propio hipervisor encendido. Esta guía explica cómo comprobarlo, apagar lo que estorba y dejar `/dev/kvm` listo.

Probado con: host Windows 11, VirtualBox, MV Ubuntu 22.04/24.04, Docker Desktop (el que exige el tutorial de clase; Docker Engine **no** necesita KVM).

---

## 1. El síntoma

Docker Desktop no arranca y, en la MV, no hay KVM:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo   # sale 0
ls -l /dev/kvm                       # No such file or directory
```

Para que funcione, el primer comando debe dar un número mayor que 0 y `/dev/kvm` debe existir.

---

## 2. Por qué pasa

VirtualBox solo puede pasar VT-x/AMD-V a la MV si **Windows no se ha quedado con el hipervisor**. Mientras Windows lo use (para WSL2, seguridad, etc.), la MV no ve las extensiones de virtualización, aunque actives la opción anidada en VirtualBox.

Para comprobarlo, en **PowerShell del host (Windows)**:

```powershell
systeminfo | Select-String -Pattern "Requisitos Hyper-V" -Context 0,1
```

Si dice **"Se detectó un hipervisor"**, Windows lo tiene encendido y hay que apagarlo (paso 4).

---

## 3. Configuración de VirtualBox (con la MV apagada)

1. En el **host**, activa la virtualización anidada (cambia el nombre por el de tu MV):

   ```powershell
   & "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "NOMBRE_DE_TU_MV" --nested-hw-virt on
   ```

2. En VirtualBox: **Configuración → Sistema → Aceleración**
   - Interfaz de paravirtualización: **KVM** (recomendado para Linux).
   - Marcar *Habilitar VT-x/AMD-V anidado* (si aparece activo, perfecto).
   - Marcar *Habilitar paginación anidada*.
3. En **Procesador**, asigna 2 o más CPU y en **Sistema** 4 GB de RAM o más (Docker Desktop es pesado).

> Si la opción *VT-x anidado* sale en gris, es la señal de que Windows tiene el hipervisor encendido: pasa al paso 4.

---

## 4. Apagar el hipervisor de Windows (host)

Hay varias cosas que lo mantienen encendido. Hay que revisar **todas**. Todo en **PowerShell del host, como administrador**.

### 4.1 Integridad de memoria
*Seguridad de Windows → Seguridad del dispositivo → Aislamiento del núcleo → Integridad de memoria → Desactivada.*

### 4.2 Plataforma de máquina virtual (WSL2 y compañía)

Comprobar:

```powershell
Get-WindowsOptionalFeature -Online | Where-Object { $_.State -eq "Enabled" -and $_.FeatureName -match "Hyper|VirtualMachine|Hypervisor|Subsystem" } | Select-Object FeatureName
```

Desactivar lo que salga (por ejemplo `VirtualMachinePlatform`):

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
```

> Esto deja sin WSL2 al host. Si lo usas, ten en cuenta que dejará de funcionar hasta que lo reactives.

### 4.3 Arranque del hipervisor

```powershell
bcdedit /set hypervisorlaunchtype off
```

### 4.4 Seguridad basada en virtualización (VBS)

Es la que casi nadie encuentra. Comprobar si está en marcha:

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Select-Object VirtualizationBasedSecurityStatus, SecurityServicesRunning
```

Si `VirtualizationBasedSecurityStatus` vale **2**, VBS está corriendo y mantiene el hipervisor aunque hagas todo lo anterior. Ver de dónde viene:

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" | Select-Object EnableVirtualizationBasedSecurity, Locked
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard" -ErrorAction SilentlyContinue
```

Si es `EnableVirtualizationBasedSecurity = 1` y no hay política que lo imponga ni `Locked`, se puede apagar:

```powershell
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Name EnableVirtualizationBasedSecurity -Value 0
bcdedit /set vsmlaunchtype off
```

### 4.5 Reiniciar el host

**Reiniciar** (no *Apagar*: el arranque rápido de Windows puede conservar el estado anterior). Después comprobar:

```powershell
systeminfo | Select-String -Pattern "Requisitos Hyper-V" -Context 0,1
```

Ya no debería poner "Se detectó un hipervisor".

---

## 5. Comprobar en la MV

Apaga la MV del todo, arráncala de nuevo y ejecuta en la **MV**:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo   # debe ser > 0
ls -l /dev/kvm                       # debe existir
```

Ejemplo de resultado correcto:

```
16
crw-rw----+ 1 root kvm 10, 232 Nov  5 13:27 /dev/kvm
```

Docker Desktop también exige que tu usuario esté en el grupo `kvm`:

```bash
sudo usermod -aG kvm $USER
```

Cierra sesión y vuelve a entrar (o reinicia la MV), y comprueba:

```bash
groups | grep kvm
```

Después sigue con la instalación de Docker Desktop del tutorial.

---

## 6. Deshacer los cambios en Windows

Si necesitas recuperar WSL2 o la seguridad original (PowerShell del host, como administrador, y reiniciar):

```powershell
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Name EnableVirtualizationBasedSecurity -Value 1
bcdedit /deletevalue vsmlaunchtype
bcdedit /set hypervisorlaunchtype auto
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
```

Y vuelve a activar *Integridad de memoria* desde Seguridad de Windows.

---

## 7. Avisos

- **Seguridad:** desactivar VBS e Integridad de memoria reduce una capa de protección de Windows. En un PC de clase conviene avisar al docente; en un equipo propio, valorar si compensa.
- **Equipos con política de dominio o de centro:** si `Locked` vale 1 o hay clave en `HKLM:\SOFTWARE\Policies\...`, la configuración la impone el administrador y puede reactivarse sola. Habrá que pedirle el cambio.
- **WSL2, Docker Desktop para Windows y Hyper-V** dejan de funcionar mientras el hipervisor esté apagado.
- **Alternativa sin tocar Windows:** instalar **Docker Engine** en la MV, que no necesita KVM. Solo sirve si el tutorial o el docente no exigen Docker Desktop.

---

## Resumen de problemas

| Síntoma | Causa | Solución |
|---|---|---|
| `egrep -c '(vmx\|svm)' /proc/cpuinfo` da 0 | Windows se queda con el hipervisor | Apartado 4 y reiniciar el host |
| Opción *VT-x anidado* en gris en VirtualBox | Idem | Apartado 4 |
| `systeminfo` sigue diciendo "Se detectó un hipervisor" tras desactivar Integridad de memoria y `hypervisorlaunchtype` | VBS sigue activa | Apartado 4.4 |
| Cambios hechos pero sin efecto | Falta reinicio real del host | Reiniciar (no Apagar) |
| `/dev/kvm` existe pero Docker Desktop se queja de permisos | Usuario fuera del grupo `kvm` | `sudo usermod -aG kvm $USER` y volver a entrar |
