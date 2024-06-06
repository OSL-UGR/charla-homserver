# Instalado de la imagen:
Lo primer que tenemos que hacer es instalar [[Ubuntu]] server sobre la tarjeta SD de la [[Raspberry|raspberry]].

Es importante remarcar que la [[Raspberry]] **NO** tiene un [[boot-loader]], esto conlleva que debemos instalar la imagen directamente sobre su memoria interna, en este caso una SD.

Por tanto los pasos son:
1. Descargamos la última versión de Ubuntu Server (en mi caso es la 22.04).
2. Nos descargamos [raspberry pi imager](https://raspberryparatorpes.net/instalacion/raspberry-pi-imager-instalador-oficial-de-sistemas-operativos/), la herramienta para montar las imágenes en la tarjeta de las [[Raspberry|raspberries]].
3. Instalamos la imagen de Ubuntu Server sobre nuestra placa con el imager.
4. Por defecto a mi no me gusta poner ningún valor de usuario y tal, en principio debería venir el usuario "ubuntu" con la contraseña "ubuntu", con permisos de sudo.
# Primeros pasos:
Hay que hacer algunas configuraciones iniciales:
1. [[#Configuración del teclado|Configuración del teclado.]]
2. [[#Creación del usuario sudoer|Crear el usuario sudoer que usaremos]].
3. [[#Borrado del usuario por defecto|Borramos el usuario "ubuntu" que se crea por defecto]].
4. [[#Configuración del ssh|Configuramos el ssh]].
5. [[#Creación de una partición SWAP|Creamos el SWAP]]
## Configuración del teclado:
Por defecto la configuración de teclado que suelen traer las imágenes es la inglesa, por tanto una de las primeras cosas que tenemos que hacer es arreglar esto:
```Bash
	sudo dpkg-reconfigure keyboard-configuration
	sudo reboot
```

Al ejecutar la primera línea vamos a abrir una pantalla que nos permitirá cambiar la distribución, y hacemos un reboot para poder actualizarlo.

>[!note] Símbolo "-"
> El símbolo - en el teclado inglés se corresponde con la ' 
> en el teclado español.

## Creación del [[Usuario]] sudoer:
También hace falta crear el usuario con el que vamos a controlar todo el dispositivo, por tanto es necesario que nuestro usuario tenga permisos de superusuario.

>[!tip] sudo
> El término **sudo** significa **superuser do**, aunque me gusta mucho más pensar que significa "super do".

Para crear un usuario y añadirlo al grupo de sudoers ejecutamos los siguientes comandos:

```shell
	sudo adduser {user}
	sudo usermod -aG sudo {user}
```

Por tanto ya tendríamos creado nuestro usuario con permisos de sudo.
## Borrado del usuario por defecto:
Para poder borrar el usuario ubuntu, que es el usuario por defecto, únicamente hace falta ejecutar un comando:
```shell
	sudo deluser --remove-home ubuntu
```

Con este comando nos aseguramos de que si existía un home asignado a ubuntu, se haya borrado también.
## Configuración del [[SSH|ssh]]:
Lo que pretendemos lograr en el ssh es permitir únicamente la conexión mediante clave privada, y no mediante contraseña, ya que es más segura, y también deshabilitar el registro desde el usuario root.

Para ello tenemos que seguir los siguientes pasos:
1. En nuestro ordenador original, tenemos que crear una llave para el ssh:
```shell
	sudo ssh-keygen
```
2. Una vez creada esta clave, la queremos mandar a la raspberry, para poder conectarnos sin contraseña.
```shell
	sudo ssh-copy-id {usuario}@{host}
```

- Tras esto tendremos que introducir la contraseña para nuestro usuario, y se guardará nuestra clave pública. 

>[!note] Alternativas
> Otra manera de hacer lo mencionado anteriormente es copiar directamente la clave pública del usuario dentro de la raspberry, pero es más complicado, sobre todo si no podemos copiar, ya que las claves suelen ser muy muy largas.

3. Una vez que hemos hecho lo anterior podemos probar a conectarnos y veremos que no requiere de contraseña.

```shell
	ssh {usuario}@{host}
```
4. Configuramos el archivo para no permitir el acceso con contraseña, ni al usuario root, y también cambiaremos el puerto para hacerlo un poquito más seguro.
## Creación de una [[Partición]] [[SWAP]]:
Como la Raspberry tiene muy poca ram, vamos a crearle una partición SWAP, para que pueda aliviar un poco la cantidad de procesos que queremos meter:
```shell
	sudo dd if=/dev/zero of=/swapfile bs=1M count=1024
	sudo mkswap /swapfile
	sudo chmod 0600 /swapfile
	sudo swapon /swapfile
```

Por último añadimos el swap al archivo fstab para que se monte automáticamente:
```sh
	sudo vim /etc/fstab
	# /swapfile  none  swap  sw  0  0
```
# Instalación de servicios:
En esta sección instalaremos Grafana con Prometheus y Mealie, para poder prestar servicios desde nuestra raspberry.

Antes de continuar vamos a actualizar nuestra máquina con:
```shell
	sudo apt-get update
	sudo apt-get upgrade
```
## Grafana + Prometheus:
### Prometheus:
Primero vamos a instalar prometheus:
```Shell
	sudo apt-get install prometheus
```

Comprobamos que esté funcionando bien y accedemos a la página:
```sh
	sudo systemctl status prometheus
```

El puerto por defecto de Prometheus es el 9090, así que accediendo a {Host}:9090 deberíamos ver la página de prometheus.


![[Pasted image 20240522135005.png]]
### Grafana:
Grafana no viene en los repositorios por defecto, así que vamos a añadir su repositorio a mano:
```sh
	wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
	sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
```

Lo que hacemos es descargarnos la clave gpg (La clave GPG es una clave que sirve para poder confirmar que lo que nos descargamos del repositorio es lo que queríamos, hay que añadirla para que no de errores).

Posterior a esto añadimos el repositorio.

Ahora tenemos que actualizar los repositorios, y podemos proceder a instalar Grafana.

```sh
	sudo apt-get update
	sudo apt-get install grafana
	sudo systemctl enable grafana-server
	sudo systemctl start grafana-server
```

Esperamos unos minutos a que se encienda, y tendremos la página principal en el puerto 3000.

![[Pasted image 20240522140427.png]]

Después de esto tenemos que logearnos con "admin" y contraseña "admin", y cambiamos la contraseña por defecto.

Añadimos nuestro prometheus a las fuentes de datos:
![[Pasted image 20240522140620.png]]

Finalmente necesitamos un dashboard, el que recomiendo personalmente es el node exporter full, que es un servicio que suele venir por defecto con prometheus, en caso contrario lo instalamos.

>[!warning] Uso de grafana en la máquina.
> Es importante remarcar como ya se menciona en algunas asignaturas de la carrera, que no es lo óptimo tener la aplicación de Grafana en la misma máquina que queremos monitorizar, ya que afecta al rendimiento, pero como va a ser un home server y suponemos que no tenemos acceso a otras máquinas, vamos a hacerlo así.

## Mealie:
Mealie es una plataforma que sirve para guardar recetas, con ingredientes, instrucciones y ese tipo de cosas.

Mealie tiene un [[Docker|docker]] compose para poder desplegarlo rápidamente, por tanto tenemos que instalar tanto [[Docker|docker]] como [[Docker Compose|docker compose]].
### Instalar docker y docker compose:
Lo primero que tenemos que hacer es instalar los repositorios de docker, para poder instalar y actualizar docker de manera cómoda. 
```sh
	sudo apt install ca-certificates curl gnupg lsb-release
	sudo mkdir /etc/apt/demokeyrings
	sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | 
	sudo gpg --dearmor -o /etc/apt/demokeyrings/demodocker.gpg
	echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/demokeyrings/demodocker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

A continuación instalamos docker y docker compose:
```sh
	sudo apt install docker docker-compose
```

Por último es recomendable añadir al usuario al grupo de docker, ya que si no tenemos que ejecutar cada comando de docker con sudo:
```sh
	sudo usermod -aG docker {user}
```

Después de esto tenemos que hacer un log out y un log in, para que se actualice. 
### Instalación de Mealie:
Primero tenemos que crear nuestro docker compose, en el que crearemos tanto una imagen de postgres como una imagen de mealie.

Por tanto nuestro archivo debe quedar parecido a:
```yml
version: "3.7"
services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:v1.6.0
    container_name: mealie
    ports:
        - "9925:9000"
    deploy:
      resources:
        limits:
          memory: 500M
    depends_on:
      - postgres
    volumes:
      - mealie-data:/app/data/
    environment:
    # Set Backend ENV Variables Here
      - ALLOW_SIGNUP=true
      - PUID=1000
      - PGID=1000
      - TZ=
      - MAX_WORKERS=1
      - WEB_CONCURRENCY=1
      - BASE_URL=

    # Database Settings
      - DB_ENGINE=postgres
      - POSTGRES_USER=mealie
      - POSTGRES_PASSWORD=
      - POSTGRES_SERVER=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=mealie
    restart: always
  postgres:
    container_name: postgres
    image: postgres:15
    restart: always
    volumes:
      - ./mealie-pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: 
      POSTGRES_USER: mealie

volumes:
  mealie-data:
    driver: local
  mealie-pgdata:
    driver: local
```

Como podemos ver faltaría ponerle una contraseña, que ya queda a elección del usuario.

Finalmente hacemos `docker-compose up` y cuando acabe de instalarse tendremos nuestro servidor de mealie funcionando.

Hemos usado esta manera de hacerlo y no otra como hacerlo desde el repositorio porque haciendo la charla pasaban cosas como:

![[Muestra mealie 1.png]]
![[Muestra mealie 2.png]]
# Conexión con internet:
Para esta
1. Comprar un dominio.
2. Establecer los DNS del dominio como los de cloudflare.
3. Crear el tunnel.
4. Añadir y enrutar los servicios.
5. Para los ssh hay que configurar el .ssh/config.
Sin embargo aquí vamos a explicar sólo los 3 últimos pasos, ya que los dos primeros son dependientes de la tecnología que se use.
## Crear el túnel:
En cloudflare la creación de túneles es muy sencilla, lo veremos con imágenes excepto en las que no hay otra opción:
1º
![[Cloudflare1.png]]
2º
![[Cloudflare2.png]]
3º
![[Cloudflare3.png]]
4º
![[Cloudflare4.png]]
5º
![[Cloudflare5.png]]
6º
![[Cloudflare6.png]]
7º
![[Cloudflare7.png]]

8º Le ponemos un nombre al túnel
![[Cloudflare8.png]]

9º En este último paso hay que aclarar que para que funcione el túnel tenemos que instalar el servicio en la raspberry, por tanto le damos a ARM y hacemos click sobre el botón de copiar, después lo pegamos en nuestra máquina y veremos cómo aparece nuestra conexión en la pestaña de connecters.
![[Cloudflare9.png]]
## Añadir y enrutar servicios:
Este paso también es muy sencillo, tenemos que decir, respecto al servidor que va a alojar el tunel, dónde está el servicio que queremos exponer, en nuestro caso va a ser localhost pero no hay que olvidar que podemos usar varias máquinas, aumentando exponencialmente el número de servicios que podemos exponer.
![[Cloudflare10.png]]
Además de al momento de la creación podemos hacerlo posteriormente:
![[Cloudflare11.png]]

![[Cloudflare12.png]]

![[Cloudflare13.png]]
Vemos que volvemos a la misma página que antes.
![[Cloudflare14.png]]
## ssh:
Para configurar el ssh, tenemos que enseñarle a nuestro ordenador a dónde tiene que apuntar cuando queramos llegar a la url, de normal habría que hacerlo a mano, pero cloudflare ofrece un demonio que lo calcula cada vez que quieras unirte, por tanto:

1. Añadir este demonio a nuestro ordenador, para que haga la función de sustituirlo.
>[!note]
>(Lo hacemos con nuestro gestor de paquetes normal, no es necesario descargar el binario para compilarlo desde 0)

2. Modificar el archivo .ssh/
![[CloudflareSSH.png]]
En este caso en lugar de ejemplo deberíamos poner nuestro dominio.
# Fuentes:
[Grafana y Prometheus](http://www.d3noob.org/2020/02/installing-prometheus-and-grafana-on.html)
[Docker y docker compose](https://www.cherryservers.com/blog/how-to-install-and-use-docker-compose-on-ubuntu-20-04)
[Mealie](https://docs.mealie.io/documentation/getting-started/installation/installation-checklist/)
