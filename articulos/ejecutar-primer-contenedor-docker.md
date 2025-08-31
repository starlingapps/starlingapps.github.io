# Ejecutar tu primer contenedor Docker

Desde la página oficial de Docker podemos ver las diferentes instalaciones de Docker para Windows, Mac OS y Linux.  
En esta ocasión vamos a instalar Docker en una distribución de Linux. La razón principal es que Docker fue desarrollado principalmente para Linux.

---

## Preparación del entorno

Yo creé una máquina virtual con **VirtualBox** y descargué la versión **Ubuntu 18.04.3 Desktop**, la cual está certificada para instalar Docker.  
Si ya tienes Linux instalado, verifica que tu distribución esté soportada.  

Opciones posibles:
- Instalar Docker directamente en Windows.
- Usar arranque dual con Ubuntu.
- Instalar Ubuntu en una máquina virtual (la opción que yo elegí para pruebas).

---

## Instalación de Docker en Ubuntu

### 1. Verificar la versión de Ubuntu

```bash
cat /etc/lsb-release
```

Resultado esperado:

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=18.04
DISTRIB_CODENAME=bionic
DISTRIB_DESCRIPTION="Ubuntu 18.04.3 LTS"
```

### 2. Actualizar el sistema operativo

```bash
sudo apt-get update
```

### 3. Instalar paquetes requeridos

```bash
sudo apt-get -y install     apt-transport-https     ca-certificates     curl     gnupg-agent     software-properties-common
```

### 4. Agregar Docker GPG Key y repositorio

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository   "deb [arch=amd64] https://download.docker.com/linux/ubuntu   $(lsb_release -cs)   stable"
```

### 5. Instalar Docker CE

```bash
sudo apt-get update
sudo apt-get install -y docker-ce=5:18.09.5~3-0~ubuntu-bionic                         docker-ce-cli=5:18.09.5~3-0~ubuntu-bionic                         containerd.io
```

> Puedes revisar otras versiones disponibles con:
>
```bash
apt-cache madison docker-ce
```

### 6. Verificar instalación

```bash
sudo docker version
```

### 7. Agregar usuario al grupo Docker

```bash
sudo usermod -a -G docker dev
```

> Esto evita tener que usar `sudo` en cada comando Docker. Después de esto, cierra y abre una nueva terminal (o reinicia la VM si es necesario).

---

## Ejecutar tu primer contenedor

```bash
docker run hello-world
```

Esto descargará la imagen `hello-world` si no existe localmente, creará un contenedor y ejecutará el servicio mostrando un mensaje en la consola.

---

## Configurar Docker Compose

Docker Compose permite levantar varios contenedores como servicios declarativos en un solo nodo.

### 1. Instalar Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version
```

### 2. Crear archivo `docker-compose.yaml`

```yaml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  redis:
    image: redis:alpine
```

### 3. Levantar los servicios

```bash
docker-compose up -d
```

Verificar contenedores en ejecución:

```bash
docker-compose ps
```

Detener los servicios:

```bash
docker-compose down
```

---

## Conclusión

Con esto ya tienes Docker y Docker Compose instalados y configurados en tu Ubuntu.  
Puedes comenzar a levantar contenedores, explorar Docker Hub y crear tus propias imágenes para proyectos reales.
