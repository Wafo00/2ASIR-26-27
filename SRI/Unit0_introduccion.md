## Distribuciones Linux oficiales de CCAA

| Comunidad autónoma	| Distribución/es asociada/s	| Base / situación aproximada
|---|---|---
| Andalucía	| Guadalinex, EducaAndOS	Ubuntu/Linux Mint | Guadalinex se discontinuó en 2018
| Aragón	| AugustuX, Colebuntu, Vitalinux	Debian/Ubuntu | varios proyectos educativos
| Asturias	| Asturix	Ubuntu | descontinuada
| Islas Baleares	| IOB	Proyecto educativo | histórico
| Canarias	| mEDUXa, Bardinux	Ubuntu/Xubuntu | proyectos educativos
| Cantabria	| LinuxGLOBAL, Labarinux	Debian/Linux Mint | históricos
| Castilla y León	| ArkiX	| Proyecto regional/educativo
| Castilla-La Mancha	| Molinux	Ubuntu | discontinuada
| Cataluña	| Linkat, CATix	Ubuntu / antiguamente openSUSE | Linkat es el proyecto educativo más conocido
| Comunidad Valenciana	| LliureX, Lazarux	Ubuntu | LliureX continúa como proyecto educativo
| Extremadura	| gnuLinEx	Debian | discontinuada
| Galicia	| Trisquel, GALPon MiniNo, Galsoft Linux, Galinux	Varias iniciativas | Trisquel es la más conocida
| Comunidad de Madrid	| MAX (Madrid Linux)	Ubuntu | orientada principalmente a educación
| Región de Murcia |	| No se consolidó una distribución autonómica propia comparable a las anteriores
| Navarra	| Proyectos basados en otras distribuciones	| No hubo una distro autonómica oficial de referencia ampliamente consolidada
| País Vasco / Euskadi |	EusLinux, EHUX	Debian/Kubuntu | principalmente educativos/universitarios
| La Rioja |	Riojix	| Proyecto histórico


## Versiones de Linux y en qué se basan:  
Ubuntu -> Debian  
Mint -> Debian  
elementaryOs -> Ubuntu-Debian  
Pop!_OS -> Ubuntu-Debian  
Zorin OS -> Ubuntu-Debian  
KDE neon -> Ubuntu -Debian  
Deepin -> Ubuntu Debian  
Kali Linux -> Debian  
Parrot OS -> Debian  
AntiX -> Debian   
MX Linux -> Debian  
Oracle Linux -> Red Hat  
CentOS Stream ->Red Hat  
Mandriva -> Red Hat  
OpenSuse -> Suse-Independiente  
Red Hat -> Fedora  
AlmaLinux -> Fedora  
ArchLinux -> Arch  
Manjaro -> Arch Linux  
EndeavourOS -> Arch  


Slackware -> Independiente  
Gentoo -> Independiente  


# Modos de conexión de la Máquina virtual  
* Por defecto: NAT  (Usa el adaptador de red de la máquina física).  
* Adaptador puente (Bridged): La máquina virtual se conecta a la tarjeta física de la máquina física, <ins>obteniendo su propia dirección IP</ins>.  
* Solo anfitrión (hostonly): Crea una red privada entre máquina física y la virtual.  
* Red interna: Crea un entorno aislado.  
* Controlador genérico: Permite usar un controlador especializado que comparte interfaz con otros modos de virtualización.  
* Red NAT: Similar al NAT, pero varias máquinas pueden estar conectadas en la misma red y pueden comunicarse. Sin acceso a internet. Aislada.  
* No conectado: Simula un cable de red desconectado.  
* Modo experimental (Cloud network): permite a la máquina virtual, entre otras cosas, escuchar y capturar todo el tráfico.    
