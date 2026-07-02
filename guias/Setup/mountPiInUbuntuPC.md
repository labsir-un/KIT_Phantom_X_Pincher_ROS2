# Montar las carpetas de la Raspberry Pi en el PC mediante SSHFS (exclusivo para Linux)

**Autor:** Johan Alejandro López Arias ([@ElJoho](https://github.com/ElJoho))

Esta guía explica cómo montar las carpetas de la Raspberry Pi del kit del PhantomX Pincher (con **Ubuntu 24.04**) dentro de un PC con **Ubuntu 24.04** usando **SSHFS**, de modo que los archivos de la Raspberry aparezcan como una carpeta local más.

> **Esta guía es exclusiva para PC con Linux (Ubuntu 24.04).** El comando `sshfs` y las opciones de montaje descritas aquí son propias de Linux. Si tu PC usa Windows, esta guía no aplica: en ese caso usa la guía de Samba o un cliente como SSHFS-Win / SFTP.

El objetivo principal es **editar los archivos de la Raspberry Pi desde Visual Studio Code en el PC sin consumir recursos en la Raspberry**. Al usar SSHFS, VS Code trata todo como si fuera local y no ejecuta ningún servidor pesado en la Raspberry Pi.

Es ideal para:

- Editar código de ROS 2 desde VS Code en el PC sin sobrecargar la Raspberry.
- Trabajar proyectos completos alojados en la Raspberry como si estuvieran en el PC.
- Navegar y abrir archivos de la Raspberry desde el explorador de archivos de Ubuntu.
- Mantener la Raspberry lo más libre de RAM y CPU posible.

---

## Datos de conexión

| Campo                                    | Valor              |
| ---------------------------------------- | ------------------ |
| **Usuario de la Raspberry Pi**           | `unpi`             |
| **IP de la Raspberry Pi**                | `192.168.10.2`     |
| **IP del PC por Ethernet**               | `192.168.10.1`     |
| **Máscara de subred**                    | `255.255.255.0`    |
| **Usuario del PC**                       | `<usuario>` (el tuyo) |
| **Carpeta de montaje en el PC**          | `~/raspberrypi`    |
| **Carpeta compartida de la Raspberry**   | `/home/unpi`       |
| **Sistema operativo de ambos equipos**   | Ubuntu 24.04 LTS   |

> En esta guía `<usuario>` representa **tu** nombre de usuario en el PC, que puede ser cualquiera. Reemplázalo por el tuyo en cada comando (puedes verlo con `whoami` o en el prompt de tu terminal).

> La Raspberry Pi ya debe tener la IP estática `192.168.10.2`. Si ya seguiste la guía de NoMachine para conexión directa por Ethernet, tu adaptador Ethernet en el PC debería estar en `192.168.10.1`. Ambos equipos quedan en la misma subred y no necesitas configurar nada más de red.

---

## ¿Cuándo usar SSHFS y cuándo Samba o Remote-SSH?

SSHFS no reemplaza a Samba ni a Remote-SSH. Cada uno resuelve un problema distinto.

| Necesitas…                                                          | Usa             |
| ------------------------------------------------------------------- | --------------- |
| Editar código en VS Code sin gastar RAM ni CPU en la Raspberry      | **SSHFS**       |
| Que las carpetas de la Raspberry aparezcan como una carpeta local   | **SSHFS**       |
| Mover archivos grandes desde el explorador de Windows               | **Samba**       |
| IntelliSense / análisis de código ejecutado en el entorno real de la Pi | **Remote-SSH** |

En resumen:

- **SSHFS**: mejor para editar archivos de la Raspberry desde el PC sin cargar la Raspberry. Es la opción de esta guía.
- **Samba**: mejor para mover archivos desde el explorador de Windows.
- **Remote-SSH**: mejor cuando necesitas que los lenguajes (C++, Python) se analicen con el entorno real de la Raspberry, **pero** ejecuta un VS Code Server en la Raspberry y consume RAM y CPU en ella.

> **Concepto importante:** Remote-SSH instala y ejecuta un servidor completo de VS Code (un proceso de Node más las extensiones) **dentro de la Raspberry Pi**. Eso es justamente el consumo de recursos que queremos evitar. SSHFS solo usa el servidor SSH/SFTP que la Raspberry ya tiene corriendo, que es muy ligero.

---

## Cómo funciona SSHFS

SSHFS es un sistema de archivos basado en **FUSE** que usa el protocolo **SFTP** (sobre SSH) para exponer una carpeta remota como si fuera local.

En este caso:

- La Raspberry Pi funciona como **servidor SSH** (ya lo tiene activo).
- El PC funciona como **cliente SSHFS**.
- El PC monta `/home/unpi` de la Raspberry en una carpeta local, por ejemplo `~/raspberrypi`.

```
PC Ubuntu (192.168.10.1)             Raspberry Pi (192.168.10.2)
─────────────────────────            ───────────────────────────
Visual Studio Code                   Servidor SSH / SFTP
Carpeta local ~/raspberrypi ◄──────►  /home/unpi/
        │                   SFTP            ▲
        └──── lee / escribe ─────────────────┘
       Conexión Ethernet directa, sin router
```

La edición, el análisis de código y VS Code se ejecutan **en el PC**. La Raspberry solo atiende peticiones SFTP livianas.

> **Concepto importante:** el comando `sshfs` siempre se ejecuta en la máquina donde quieres que **aparezca** la carpeta (el PC), y se conecta a la máquina cuyos archivos quieres ver (la Raspberry). Nunca entres por SSH a la Raspberry para montar: el objetivo es traer los archivos de la Raspberry **hacia** el PC.

---

## Requisitos previos

Antes de empezar, asegúrate de tener:

- Raspberry Pi 5 encendida con Ubuntu 24.04 LTS.
- PC con Ubuntu 24.04 LTS.
- Conexión de red entre ambos (por ejemplo cable Ethernet directo).
- IP de la Raspberry Pi: `192.168.10.2`.
- IP del adaptador Ethernet del PC: `192.168.10.1`.
- Servidor SSH activo en la Raspberry Pi.

---

## 1. Verificar que SSH está activo en la Raspberry Pi

SSHFS necesita que el servidor SSH de la Raspberry esté corriendo. Para confirmarlo, en la terminal de la Raspberry ejecuta:

```
sudo systemctl status ssh
```

Debe aparecer:

```
Active: active (running)
```

Presiona `q` para salir. Si quieres una respuesta rápida de sí/no:

```
systemctl is-active ssh      # imprime "active" o "inactive"
systemctl is-enabled ssh     # imprime "enabled" (arranca al encender) o "disabled"
```

Lo ideal es que ambos digan `active` y `enabled`, para que el montaje siga funcionando después de reiniciar la Raspberry.

Si SSH no estuviera instalado o activo, en la Raspberry Pi:

```
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

Anota la IP de la Raspberry con:

```
hostname -I
```

---

## 2. Probar la conexión SSH desde el PC

Antes de montar nada, confirma que el PC puede llegar a la Raspberry. **Este comando se ejecuta en el PC:**

```
ssh unpi@192.168.10.2
```

La primera vez aparecerá un mensaje sobre la autenticidad del host y la huella ED25519. Escribe `yes`. Luego introduce la contraseña de `unpi`.

Si entras correctamente, verás el prompt cambiar a:

```
unpi@unpi:~$
```

Eso significa que ya estás **dentro de la Raspberry Pi**. Para volver al PC:

```
exit
```

El prompt debe volver a algo como:

```
<usuario>@<pc>:~$
```

> **Muy importante:** fíjate siempre en el prompt. Si dice `unpi@unpi` estás en la Raspberry; si muestra tu usuario del PC (`<usuario>@<pc>`) estás en el PC. Los comandos de montaje de las secciones siguientes van **en el PC** (`<usuario>@<pc>`), no en la Raspberry.

---

## 3. Instalar SSHFS en el PC

**En el PC** (no en la Raspberry), instala SSHFS:

```
sudo apt update
sudo apt install sshfs
```

---

## 4. Crear la carpeta de montaje en el PC

**En el PC**, crea la carpeta local donde aparecerán los archivos de la Raspberry:

```
mkdir -p ~/raspberrypi
```

Esta carpeta estará vacía hasta que se realice el montaje.

---

## 5. Montar la carpeta de la Raspberry en el PC

**En el PC**, ejecuta el montaje:

```
sshfs unpi@192.168.10.2:/home/unpi ~/raspberrypi \
  -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,follow_symlinks
```

Introduce la contraseña de `unpi` cuando la pida.

Después de esto, `~/raspberrypi` en el PC mostrará el contenido de `/home/unpi` de la Raspberry. Verifícalo con:

```
ls ~/raspberrypi
```

---

## 6. Explicación de las opciones de montaje

| Opción                     | Función                                                                 |
| -------------------------- | ----------------------------------------------------------------------- |
| `unpi@192.168.10.2:/home/unpi` | Usuario, IP y carpeta remota de la Raspberry que se va a montar.    |
| `~/raspberrypi`            | Carpeta local del PC donde aparecerá el contenido remoto.               |
| `reconnect`                | Reintenta la conexión automáticamente si se cae (útil en Wi-Fi).        |
| `ServerAliveInterval=15`   | Envía una señal cada 15 s para mantener viva la sesión SSH.             |
| `ServerAliveCountMax=3`    | Número de señales sin respuesta antes de considerar caída la conexión.  |
| `follow_symlinks`          | Sigue los enlaces simbólicos de la Raspberry como si fueran archivos.   |

> Las opciones `reconnect` y `ServerAliveInterval` son las que evitan que el montaje se quede "colgado" cuando la red tiene pequeños cortes, especialmente sobre Wi-Fi.

---

## 7. Abrir la carpeta en Visual Studio Code

Con el montaje activo, abre la carpeta en VS Code **en el PC**:

```
code ~/raspberrypi
```

O desde VS Code: **File → Open Folder → `~/raspberrypi`**.

VS Code tratará todo como local: no instala ningún servidor en la Raspberry, no ejecuta el host de extensiones remoto y no compila nada en ella. Toda la carga queda en el PC, que es justamente el objetivo.

> **Nota sobre latencia:** SSHFS tiene más latencia que un disco local. Las operaciones que tocan muchos archivos a la vez (un `git status` grande, búsquedas en todo el proyecto, indexado) se sentirán más lentas que editando directamente en la Raspberry. Ese es el precio de liberar de trabajo a la Raspberry.

---

## 8. Desmontar la carpeta

Cuando termines, desmonta la carpeta **en el PC**:

```
fusermount -u ~/raspberrypi
```

La carpeta `~/raspberrypi` volverá a quedar vacía, pero seguirá existiendo para el próximo montaje.

---

## 9. (Opcional) Login sin contraseña con llaves SSH

Para no escribir la contraseña en cada montaje, genera una llave SSH **en el PC** (si no la tienes) y cópiala a la Raspberry:

```
ssh-keygen -t ed25519
ssh-copy-id unpi@192.168.10.2
```

A partir de ahí, el montaje no volverá a pedir contraseña.

---

## 10. (Opcional) Montaje automático al arrancar el PC

Si quieres que la carpeta se monte sola al iniciar sesión, y ya configuraste las llaves SSH del paso anterior, edita el archivo `/etc/fstab` **en el PC**:

```
sudo nano /etc/fstab
```

Añade esta línea (ajusta el usuario del PC y la ruta de la llave):

```
unpi@192.168.10.2:/home/unpi  /home/<usuario>/raspberrypi  fuse.sshfs  _netdev,user,idmap=user,reconnect,ServerAliveInterval=15,IdentityFile=/home/<usuario>/.ssh/id_ed25519,allow_other,default_permissions  0  0
```

Guarda y cierra (`Ctrl + O`, `Enter`, `Ctrl + X`).

---

## 11. Verificación final

Con la carpeta montada, desde el **PC**:

```
touch ~/raspberrypi/prueba_desde_pc.txt
```

Desde la **Raspberry Pi**, verifica que el archivo aparece en `/home/unpi`:

```
ls -l /home/unpi
```

Ahora al revés. Desde la **Raspberry Pi**:

```
touch /home/unpi/prueba_desde_raspberry.txt
```

Desde el **PC**, verifica que el archivo aparece en la carpeta montada:

```
ls -l ~/raspberrypi
```

Si puedes crear archivos desde ambos lados y se ven en el otro, SSHFS quedó funcionando correctamente.

---


# Resumen rápido

| Acción                                     | Comando / valor                                                                 |
| ------------------------------------------ | ------------------------------------------------------------------------------- |
| IP de la Raspberry Pi                      | `192.168.10.2`                                                                  |
| IP del PC por Ethernet                     | `192.168.10.1`                                                                  |
| Usuario de la Raspberry                    | `unpi`                                                                          |
| Carpeta remota                             | `/home/unpi`                                                                    |
| Carpeta de montaje en el PC                | `~/raspberrypi`                                                                 |
| Verificar SSH en la Raspberry              | `sudo systemctl status ssh`                                                     |
| Probar conexión desde el PC                | `ssh unpi@192.168.10.2`                                                         |
| Instalar SSHFS (en el PC)                  | `sudo apt install sshfs`                                                        |
| Crear carpeta de montaje (en el PC)        | `mkdir -p ~/raspberrypi`                                                        |
| Montar (en el PC)                          | `sshfs unpi@192.168.10.2:/home/unpi ~/raspberrypi -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,follow_symlinks` |
| Abrir en VS Code                           | `code ~/raspberrypi`                                                            |
| Desmontar (en el PC)                       | `fusermount -u ~/raspberrypi`                                                   |
| Login sin contraseña                       | `ssh-keygen -t ed25519` + `ssh-copy-id unpi@192.168.10.2`                       |
| Limpiar auto-montaje en la Raspberry       | `fusermount -u ~/raspberrypi` + `rmdir ~/raspberrypi`                           |

---

# Recursos

- Documentación oficial de SSHFS (proyecto libfuse).
- Documentación de Ubuntu sobre FUSE y SSHFS.
- Documentación de OpenSSH.