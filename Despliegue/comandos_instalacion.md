0. Comandos de instalación y despliegue

Este documento detalla, en orden exacto, los comandos utilizados para desplegar Calibre-Web sobre Podman en una instancia EC2 de AWS. Cualquier persona que copie estos comandos en el mismo orden debería poder levantar un entorno idéntico.

Requisitos previos: tener Podman instalado en la máquina local y en el servidor remoto, acceso SSH a la instancia (`Proyect16.pem`), y permisos de sudo en el servidor.


1. Empaquetar la imagen en la máquina local (Windows)

Exporta la imagen de Docker Hub a un archivo `.tar` portable, para no depender de la conectividad del servidor remoto hacia el registro de imágenes.

```powershell
PS C:\Users\ASUS INTEL CORE i7\OneDrive\Escritorio> podman save -o .\calibre_web.tar docker.io/linuxserver/calibre-web
```

Esto genera `calibre_web.tar` en el Escritorio, con la imagen completa (capas + metadatos) lista para transferir.


2. Transferir la imagen al servidor (EC2)

```bash
scp -i "proyect16.pem" .\calibre_web.tar ubuntu@18.188.134.158:/home/ubuntu/
```

Sube la imagen empaquetada desde la máquina local a la instancia EC2.


3. Mover el archivo al usuario sin privilegios

```bash
sudo mv /home/ubuntu/calibre_web.tar /home/os_user/
```

Mueve el archivo al directorio del usuario "os_user", que es el usuario sin privilegios (no root) bajo el cual correrá Podman en modo rootless.


4. Asignar permisos al usuario "os_user"

```bash
sudo chown os_user:os_user /home/os_user/calibre_web.tar
```

Asigna los permisos correctos al usuario "os_user" sobre el archivo recién movido.

5. Cargar la imagen desde el `.tar`

```bash
podman load -i ~/calibre_web.tar
```
Carga la imagen de Docker directamente desde el archivo `.tar`, sin necesidad de descargarla nuevamente de internet.


6. Crear los directorios para volúmenes persistentes

```bash
mkdir -p ~/proyecto_calibre/config
```

Crea los directorios que se usarán como volúmenes persistentes para la configuración de la aplicación.


7. Mapear permisos del host al usuario interno del contenedor

```bash
podman unshare chown -R 911:911 ~/proyecto_calibre/config
```

Mapea los permisos del host al UID/GID interno (`911:911`) que usa el proceso dentro del contenedor de `linuxserver/calibre-web`, evitando problemas de permisos por el uso de *user namespaces* en modo rootless.


8. Instalar Calibre en el host

```bash
sudo apt update
sudo apt install -y calibre
```

Instala las herramientas de Calibre directamente en el host, necesarias para interactuar con la base de datos de la biblioteca desde fuera del contenedor.

9. Desplegar el contenedor con límites de cgroups

```bash
podman run -d --name calibre_app -p 8083:8083 \
  -v ~/proyecto_calibre/config:/config:Z \
  -v ~/proyecto_calibre/libros:/books:Z \
  --memory="512m" --memory-swap="768m" --cpus="0.5" \
  docker.io/linuxserver/calibre-web
```

Despliega la aplicación en segundo plano, expone el puerto `8083`, monta los volúmenes persistentes de configuración y libros, y limita el uso de recursos del contenedor a **0.5 CPUs** y **512 MB de RAM** (con hasta 768 MB de swap) mediante **cgroups**.


10. Verificación rápida

Para confirmar que el contenedor quedó corriendo correctamente:

```bash
podman ps
podman logs calibre_app
```

Y para acceder a la aplicación desde el navegador:

```
http://3.148.69.87:8083
http://calibreesan.xyz
http://calibreesan.xyz:8083
```
