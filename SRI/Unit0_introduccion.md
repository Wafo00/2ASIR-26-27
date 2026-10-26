# Unidad 0: Introducción

## Índice

- [Distribuciones Linux oficiales de las comunidades autónomas](#distribuciones-linux-oficiales-de-las-comunidades-autónomas)
- [Distribuciones de Linux y en qué se basan](#distribuciones-de-linux-y-en-qué-se-basan)
- [Modos de conexión de la máquina virtual](#modos-de-conexión-de-la-máquina-virtual)

## Distribuciones Linux oficiales de las comunidades autónomas

| Comunidad autónoma | Distribución/es asociada/s | Base | Situación aproximada |
|---|---|---|---|
| Andalucía | Guadalinex, EducaAndOS | Ubuntu / Linux Mint | Guadalinex se discontinuó en 2018 |
| Aragón | AugustuX, Colebuntu, Vitalinux | Debian / Ubuntu | Varios proyectos educativos |
| Asturias | Asturix | Ubuntu | Descontinuada |
| Islas Baleares | IOB | | Proyecto educativo, histórico |
| Canarias | mEDUXa, Bardinux | Ubuntu / Xubuntu | Proyectos educativos |
| Cantabria | LinuxGLOBAL, Labarinux | Debian / Linux Mint | Históricos |
| Castilla y León | ArkiX | | Proyecto regional/educativo |
| Castilla-La Mancha | Molinux | Ubuntu | Discontinuada |
| Cataluña | Linkat, CATix | Ubuntu / antiguamente openSUSE | Linkat es el proyecto educativo más conocido |
| Comunidad Valenciana | LliureX, Lazarux | Ubuntu | LliureX continúa como proyecto educativo |
| Extremadura | gnuLinEx | Debian | Discontinuada |
| Galicia | Trisquel, GALPon MiniNo, Galsoft Linux, Galinux | Varias iniciativas | Trisquel es la más conocida |
| Comunidad de Madrid | MAX (Madrid Linux) | Ubuntu | Orientada principalmente a educación |
| Región de Murcia | | | No se consolidó una distribución autonómica propia comparable a las anteriores |
| Navarra | | Proyectos basados en otras distribuciones | No hubo una distro autonómica oficial de referencia ampliamente consolidada |
| País Vasco / Euskadi | EusLinux, EHUX | Debian / Kubuntu | Principalmente educativos/universitarios |
| La Rioja | Riojix | | Proyecto histórico |

## Distribuciones de Linux y en qué se basan

| Distribución | Se basa en |
|---|---|
| Ubuntu | Debian |
| Mint | Debian |
| elementary OS | Ubuntu - Debian |
| Pop!_OS | Ubuntu - Debian |
| Zorin OS | Ubuntu - Debian |
| KDE neon | Ubuntu - Debian |
| Deepin | Ubuntu - Debian |
| Kali Linux | Debian |
| Parrot OS | Debian |
| AntiX | Debian |
| MX Linux | Debian |
| Oracle Linux | Red Hat |
| CentOS Stream | Red Hat |
| Mandriva | Red Hat |
| openSUSE | SUSE (independiente) |
| Red Hat | Fedora |
| AlmaLinux | Fedora |
| Arch Linux | Arch |
| Manjaro | Arch Linux |
| EndeavourOS | Arch |
| Slackware | Independiente |
| Gentoo | Independiente |

## Modos de conexión de la máquina virtual

| Modo | Qué hace |
|---|---|
| NAT (por defecto) | Usa el adaptador de red de la máquina física |
| Adaptador puente (*bridged*) | La máquina virtual se conecta a la tarjeta física de la máquina física y obtiene **su propia dirección IP** |
| Solo anfitrión (*host-only*) | Crea una red privada entre la máquina física y la virtual |
| Red interna | Crea un entorno aislado |
| Controlador genérico | Permite usar un controlador especializado que comparte interfaz con otros modos de virtualización |
| Red NAT | Similar al NAT, pero varias máquinas pueden estar conectadas en la misma red y comunicarse entre sí. Sin acceso a internet. Aislada |
| No conectado | Simula un cable de red desconectado |
| Modo experimental (*cloud network*) | Permite a la máquina virtual, entre otras cosas, escuchar y capturar todo el tráfico |
