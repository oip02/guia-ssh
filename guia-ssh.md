# Guía de instalación y configuración de SSH en Ubuntu Server

Esta guía explica cómo **instalar un servidor SSH** en Ubuntu Server y cómo **conectarse a él desde Windows**. La he escrito a partir de la práctica que hice en mi propio laboratorio: una máquina virtual con Ubuntu Server en *VMware Workstation*, administrada desde la PowerShell de mi PC con Windows. Todas las capturas y direcciones son reales.

## Índice

- [1. Introducción](#1-introducción)
- [2. Mi laboratorio y requisitos previos](#2-mi-laboratorio-y-requisitos-previos)
- [3. Procedimiento](#3-procedimiento)
  - [3.1 Preparar la red de la máquina virtual](#31-preparar-la-red-de-la-máquina-virtual)
  - [3.2 Instalar el servidor OpenSSH](#32-instalar-el-servidor-openssh)
  - [3.3 Comprobar y activar el servicio](#33-comprobar-y-activar-el-servicio)
  - [3.4 Averiguar la IP del servidor](#34-averiguar-la-ip-del-servidor)
  - [3.5 Conectarse desde Windows](#35-conectarse-desde-windows)
- [4. Comprobación](#4-comprobación)
- [5. Resolución de problemas](#5-resolución-de-problemas)
  - [5.1 Lo que me pasó: la IP cambió por DHCP](#51-lo-que-me-pasó-la-ip-cambió-por-dhcp)
  - [5.2 Comprobar que el servidor escucha en el puerto 22](#52-comprobar-que-el-servidor-escucha-en-el-puerto-22)
  - [5.3 Nota sobre PowerShell](#53-nota-sobre-powershell)
- [6. Copiar archivos con SCP y SFTP](#6-copiar-archivos-con-scp-y-sftp)
  - [6.1 SCP: copias rápidas en un solo comando](#61-scp-copias-rápidas-en-un-solo-comando)
  - [6.2 SFTP: sesión interactiva](#62-sftp-sesión-interactiva)
- [7. Referencias](#7-referencias)

---

## 1. Introducción

**SSH** (*Secure Shell*) es un protocolo para administrar un equipo a distancia a través de la red. Todo lo que viaja entre el cliente y el servidor (usuario, contraseña y comandos) va **cifrado**. Por eso sustituye a protocolos antiguos como *Telnet*, que enviaban la información en texto plano.

El servidor SSH escucha por defecto en el **puerto 22/TCP**. En Ubuntu, el programa servidor se llama `sshd` y viene en el paquete `openssh-server`.

---

## 2. Mi laboratorio y requisitos previos

| Elemento | Mi equipo | Para qué lo uso |
|---|---|---|
| Virtualizador | VMware Workstation | Crear la máquina virtual |
| Servidor | Ubuntu Server 26.04.1 LTS, 6 GB de RAM, 2 CPU, disco de 20 GB | Equipo al que me conecto |
| Nombre y usuario del servidor | `erver` / `oip` | Datos para iniciar sesión |
| IP del servidor | `192.168.1.34/24` al principio; después `192.168.1.136/24` (por DHCP, interfaz `ens33`, ver [5.1](#51-lo-que-me-pasó-la-ip-cambió-por-dhcp)) | Dirección a la que me conecto |
| Cliente | Windows con PowerShell 5.1 | Equipo desde el que administro |

Para repetir la práctica necesitas:

- Una máquina virtual con **Ubuntu Server** instalado.
- Un usuario con permisos de administrador (`sudo`).
- Conexión a Internet en la máquina virtual para descargar paquetes.
- Un equipo Windows o Linux en la misma red, que hará de cliente.

> **Aviso:** la tarjeta de red de la máquina virtual tiene que estar en modo **puente** (*bridged*). Si se deja en *NAT*, la máquina recibe una IP de una red interna de VMware y es más difícil llegar a ella desde otros equipos.

---

## 3. Procedimiento

### 3.1 Preparar la red de la máquina virtual

1. En VMware Workstation, abre la **Configuración de máquina virtual**.
2. Selecciona **Adaptador de red** y marca **Conexión en puente: conectado directamente a la red física**.
3. Durante la instalación de Ubuntu Server, en la pantalla *Network configuration*, comprueba que la interfaz `ens33` recibe una IP por DHCP de tu red. En mi caso fue `192.168.1.34/24`.

![Adaptador de red en modo puente y la IP que recibe la interfaz ens33](imagenes/01-vmware-red-bridged.png)

### 3.2 Instalar el servidor OpenSSH

Con tu usuario normal (no como *root*), actualiza la lista de paquetes e instala el servidor:

```bash
sudo apt update
sudo apt install openssh-server
```

Además de `openssh-server`, se instalan dos dependencias: `libwrap0` y `openssh-sftp-server`. Esta última permite también copiar archivos de forma segura con **SFTP**.

![Instalación del paquete openssh-server](imagenes/02-apt-install.png)

### 3.3 Comprobar y activar el servicio

Comprueba el estado con `systemctl status ssh`. Al hacerlo nada más instalar, a mí me apareció así:

![El servicio aparece como inactive (dead) y disabled](imagenes/03a-systemctl-status-inactive.png)

Parece un error, pero no lo es. En las versiones recientes de Ubuntu, SSH se arranca **por socket**: la línea `TriggeredBy: ssh.socket` indica que `ssh.socket` vigila el puerto 22 y arranca el servicio en cuanto llega una conexión.

Para que el servicio quede siempre en marcha y se inicie con el sistema:

```bash
sudo systemctl enable --now ssh
systemctl status ssh
```

Ahora el estado es `active (running)` y el registro muestra que `sshd` escucha en el puerto 22, tanto en IPv4 (`0.0.0.0`) como en IPv6 (`::`).

![El servicio activo y escuchando en el puerto 22](imagenes/03b-systemctl-status-active.png)

### 3.4 Averiguar la IP del servidor

Para conectarte necesitas la IP del servidor. Obtenla con `ip a` y busca la línea `inet` de la interfaz de red (no la de `lo`, que es la interfaz local):

![Salida de ip a con la IP 192.168.1.34 en ens33](imagenes/04-ip-a.png)

En mi caso, la interfaz `ens33` tiene la IP **192.168.1.34** con máscara `/24`.

### 3.5 Conectarse desde Windows

Windows 10 y 11 ya traen el cliente SSH, así que no hace falta instalar nada.

1. Abre **PowerShell**.
2. Escribe `ssh` seguido de tu usuario del servidor, una arroba y la IP:

```powershell
ssh oip@192.168.1.34
```

3. La primera vez, el cliente no conoce al servidor y enseña su **huella digital** (*fingerprint*). Escribe `yes` para aceptarla:

![Primera conexión: el cliente pide confirmar la huella ED25519 del servidor](imagenes/05-primera-conexion.png)

4. El cliente guarda la huella en el archivo `known_hosts`. Así, las siguientes veces comprobará que el servidor es el mismo y avisará si cambia.
5. Escribe la contraseña de tu usuario. Mientras la escribes no aparece nada en pantalla, y es normal.

---

## 4. Comprobación

Si la conexión funciona, verás el mensaje de bienvenida de Ubuntu y el *prompt* cambiará al del servidor (`oip@erver:~$`). Para confirmar dónde estás conectado:

```bash
whoami
hostname
```

![Sesión iniciada en el servidor desde PowerShell](imagenes/06-sesion-dentro.png)

Para cerrar la sesión remota y volver a PowerShell, escribe `exit`.

---

## 5. Resolución de problemas

Estos son los problemas que me encontré al hacer la práctica, y otros habituales:

| Problema | Causa | Solución |
|---|---|---|
| Instalé el paquete como *root* con `sudo su` | Trabajar como *root* no es buena práctica y me despisté | Desinstalé con `sudo apt purge openssh-server`, salí de *root* con `exit` y lo repetí con mi usuario y `sudo` |
| El nombre del servidor sale como `erver` | Errata al escribir el nombre durante la instalación | No afecta a SSH. Se puede cambiar con `sudo hostnamectl set-hostname server` |
| `systemctl status ssh` dice `inactive (dead)` | Ubuntu arranca SSH por socket (`ssh.socket`) | No es un fallo. Con `sudo systemctl enable --now ssh` queda siempre activo |
| `Connection timed out` | IP equivocada o máquina virtual en NAT | Revisa la IP con `ip a` y pon la red en modo puente |
| `Connection refused` | El servidor SSH no está instalado o está parado | `sudo systemctl start ssh` |
| `Permission denied` | Usuario o contraseña incorrectos | Comprueba el usuario (`oip` en mi caso) y vuelve a escribir la contraseña |
| Al día siguiente no conectaba: `Connection timed out` | El router le dio **otra IP por DHCP** a la máquina virtual (pasó de `.34` a `.136`) | Mirar la IP nueva con `ip a` en la consola de VMware y conectar a esa |
| `REMOTE HOST IDENTIFICATION HAS CHANGED!` | Se reinstaló el servidor o es otra máquina con la misma IP, y la huella guardada en `known_hosts` ya no coincide | Si sabes que el cambio es legítimo, borra la huella antigua con `ssh-keygen -R 192.168.1.136` y vuelve a conectar |


### 5.1 Lo que me pasó: la IP cambió por DHCP

Al volver a la práctica otro día, `ssh` dejaba de responder. Entré en la consola de la máquina virtual en VMware, ejecuté `ip a` y vi que la interfaz `ens33` tenía ahora la IP **192.168.1.136**:

![La interfaz ens33 con la nueva IP 192.168.1.136](imagenes/08-ip-cambiada-dhcp.png)

Al conectar a la IP nueva, SSH me avisó de que la huella del servidor ya la conocía con otra dirección (`known_hosts:1: 192.168.1.34`). Como era el mismo servidor, la acepté con `yes`.

> **Cómo evitarlo:** en un servidor real conviene darle una **IP fija**, ya sea con una reserva DHCP en el router o configurándola en Netplan (`/etc/netplan/`).

### 5.2 Comprobar que el servidor escucha en el puerto 22

Si dudas de si el servicio está escuchando, ejecuta en el servidor:

```bash
sudo ss -tlnp | grep :22
```

Debe aparecer una línea con `LISTEN` en `0.0.0.0:22` (y en `[::]:22` para IPv6). Si no sale nada, el servicio no está escuchando y hay que revisar `systemctl status ssh`.

### 5.3 Nota sobre PowerShell

En la práctica abrí PowerShell **como administrador**, pero para usar `ssh` o `scp` no hace falta: basta con abrirlo de forma normal. Trabajar con los mínimos permisos necesarios es una buena práctica de seguridad.

---

## 6. Copiar archivos con SCP y SFTP

Al instalar `openssh-server` también se instaló `openssh-sftp-server`, así que con el mismo usuario y contraseña de SSH se pueden **copiar archivos de forma cifrada** entre Windows y el servidor.

### 6.1 SCP: copias rápidas en un solo comando

| Qué quiero hacer | Comando (desde PowerShell) | Resultado |
|---|---|---|
| Subir un archivo al servidor | `scp .\prueba.txt oip@192.168.1.136:/home/oip/` | El archivo queda en mi carpeta personal de `erver` |
| Bajar un archivo del servidor | `scp oip@192.168.1.136:/etc/hostname .\hostname-servidor.txt` | Copia el archivo a la carpeta actual de Windows |
| Copiar una carpeta entera | `scp -r .\carpeta oip@192.168.1.136:/home/oip/` | La opción `-r` copia también el contenido |

Así lo comprobé en mi laboratorio. Como la máquina virtual había cambiado de IP (ahora `192.168.1.136`), SSH me avisó de que esa misma huella ya la conocía con la dirección `192.168.1.34` y tuve que confirmarla otra vez con `yes`:

![Copia de archivos con scp entre Windows y el servidor](imagenes/07-scp.png)

### 6.2 SFTP: sesión interactiva

Con `sftp oip@192.168.1.136` se abre una sesión en la que se puede navegar y copiar varios archivos:

```text
sftp> ls          lista los archivos del servidor
sftp> put archivo sube un archivo
sftp> get archivo descarga un archivo
sftp> bye         cierra la sesión
```

> **Consejo:** en la ruta del servidor siempre van los dos puntos después de la IP (`oip@192.168.1.136:/ruta`). Si se olvidan, `scp` hace una copia local en Windows en vez de enviar el archivo.

---

## 7. Referencias

- [Documentación de Ubuntu Server sobre OpenSSH](https://ubuntu.com/server/docs/openssh-server)
- [Página del proyecto OpenSSH](https://www.openssh.com/)
- [Cliente OpenSSH en Windows (Microsoft Learn)](https://learn.microsoft.com/es-es/windows-server/administration/openssh/openssh_overview)
