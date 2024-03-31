# Guía rápida de configuración de Wordpress con SSL usando docker compose.

## Introducción

Wordpress es un CMS gratuito y de código abierto construido con PHP que usa MySQL como base de datos.
Permite su administración a través de una interfaz web, siendo esta una de las razones de la popularidad de Wordpress para creación de sitios web, los cuales van desde blogs, hasta páginas de productos y sitios de comercio electrónico.

La instalación clásica de Wordpress puede llevar mucho tiempo, por eso en esta guía veremos como agilizar su instalación mediante el uso de herramientas como Docker y Docker Compose.

Construiremos los contenedores necesarios para realizar el despliegue de Wordpress en un entorno de producción, esto incluirá la configuración de un certificado SSL para el uso de HTTPS.

Esta guía esta basada en un artículo realizado por DigitalOcean, el link al artículo original lo puedes encontrar al final de esta guía.

## Prerrequisitos

Para realizar el proceso de despliegue se necesitará:

- Servidor Linux 
- Docker
- Docker compose
- Instalación de NginxProxyManager (recomendado)
- Dominio propio (en adelante referido como: **dominio**)
- Puertos 80 y 443 habilitados en UFW

## Configuración de dominio

Crear dos registros de tipo 'A' que apunten a la IP pública del servidor:

- dominio
- www.dominio

Una vez configurado, comenzamos.

## Despliegue de Wordpress con Docker Compose

### Recursos necesarios

Este repositorio incluye lo necesario para realizar el despliegue, lo descargaremos:

~~~
$ git clone https://github.com/MarcoBardalesRodriguez/wordpress-dockercompose.git 
~~~

La carpeta **wordpress-dockercompose/files/** tiene las configuraciones que iremos requiriendo durante esta guía.

### Creación de directorios necesarios

Ingresamos al directorio para nuestro proyecto, en este caso, se llama: **wordpress-dockercompose**

~~~
$ cd wordpress-dockercompose
~~~

Y creamos las siguientes carpetas:

~~~
$ mkdir nginx-conf
$ mkdir wordpress-data
$ mkdir db-data
~~~

### Paso 1: Despliegue HTTP

#### Archivos necesarios:
Copiaremos los archivos necesarios desde la carpeta **files**:

~~~
$ cp ./files/step1/nginx.conf ./nginx-conf/
$ cp ./files/step1/.env ./
$ cp ./files/step1/docker-compose.yml ./
~~~

**Importante**, reemplazar en los archivos **nginx.conf** y **docker-compose.yml** la palabra ***dominio*** por el dominio que se usará para el despliegue, y agregar un email valido en el servicio ***certbot*** del archivo **docker-compose.yml**:

~~~
...
  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - wordpress-certbot:/etc/letsencrypt
      - wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email your@email --agree-tos --no-eff-email --staging -d dominio -d www.dominio
...
~~~

#### Configurar puertos para Wordpress

***Si usa NginxProxyManager:***

Configurar nuevo proxy host

~~~
Domain Names: dominio www.dominio
Scheme: http
IP: tu-ip
Port: 8280
~~~

***Si no usa NginxProxyManager:***

Habilitar puerto 80 en UFW.

Modificar archivo **docker-compose.yml** y cambiar el puerto de salida del servicio **webserver**:

~~~
...
  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - wordpress-certbot:/etc/letsencrypt
    networks:
      - wordpress-network
...
~~~

#### Obtener SSL

Iniciar contenedores

~~~
$ docker compose up -d
~~~

Revisar el estado de los contenedores

~~~
$ docker compose ps
~~~

La salida mostrara algo semejante a:

~~~
  Name                 Command               State           Ports
-------------------------------------------------------------------------
certbot     certbot certonly --webroot ...   Exit 0
db          docker-entrypoint.sh --def ...   Up       3306/tcp, 33060/tcp
webserver   nginx -g daemon off;             Up       0.0.0.0:80->80/tcp
wordpress   docker-entrypoint.sh php-fpm     Up       9000/tcp
~~~

Revisar que los certificados ssl se hayan obtenido correctamente:

~~~
$ docker compose exec webserver ls -la /etc/letsencryt/live
~~~

La salida debe contener el nombre de su dominio en el listado:

~~~
total 16
drwx------    3 root     root          4096 Mar 31 15:45 .
drwxr-xr-x    9 root     root          4096 Mar 31 15:45 ..
-rw-r--r--    1 root     root           740 Mar 31 15:45 README
drwxr-xr-x    2 root     root          4096 Mar 31 15:45 dominio
~~~


### Paso 2: Certificados SSL

En el archivo **docker-compose.yml** modificaremos el servicio ***certbot***, en la instruccion ***command***, reemplazamos
***--staging*** por ***--force-renewal*** :

~~~
...
  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      - wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email your@email --agree-tos --no-eff-email --force-renewal -d dominio -d www.dominio
...
~~~

**Importante**, no olvidar reemplazar tanto ***dominio*** como ***your@email*** con los datos respectivos.


#### Recreamos el servicio **certbot**

Ejecutamos el siguiente comando:

~~~
$ docker compose up --force-recreate --no-deps certbot
~~~

Esperamos una salida semejante a:

~~~
Recreating certbot ... done
Attaching to certbot
certbot      | Saving debug log to /var/log/letsencrypt/letsencrypt.log
certbot      | Plugins selected: Authenticator webroot, Installer None
certbot      | Renewing an existing certificate
certbot      | Performing the following challenges:
certbot      | http-01 challenge for dominio
certbot      | http-01 challenge for www.dominio
certbot      | Using the webroot path /var/www/html for all unmatched domains.
certbot      | Waiting for verification...
certbot      | Cleaning up challenges
certbot      | IMPORTANT NOTES:
certbot      |  - Congratulations! Your certificate and chain have been saved at:
certbot      |    /etc/letsencrypt/live/dominio/fullchain.pem
certbot      |    Your key file has been saved at:
certbot      |    /etc/letsencrypt/live/dominio/privkey.pem
certbot      |    Your cert will expire on 2019-08-08. To obtain a new or tweaked
certbot      |    version of this certificate in the future, simply run certbot
certbot      |    again. To non-interactively renew *all* of your certificates, run
certbot      |    "certbot renew"
certbot      |  - Your account credentials have been saved in your Certbot
certbot      |    configuration directory at /etc/letsencrypt. You should make a
certbot      |    secure backup of this folder now. This configuration directory will
certbot      |    also contain certificates and private keys obtained by Certbot so
certbot      |    making regular backups of this folder is ideal.
certbot      |  - If you like Certbot, please consider supporting our work by:
certbot      |
certbot      |    Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
certbot      |    Donating to EFF:                    https://eff.org/donate-le
certbot      |
certbot exited with code 0
~~~

### Paso 3: Despliegue HTTPS

#### Detenemos el contenedor del servidor web

Ejecutamos el siguiente comando:

~~~
$ docker compose stop webserver
~~~

#### Archivos necesarios

Descargamos un archivo de configuración usando curl:

~~~
$ curl -sSLo nginx-conf/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
~~~

O podemos copiarlo desde la carpeta **files**:

~~~
$ cp ./files/step3/options-ssl-nginx.conf ./nginx-conf/
~~~

#### Configuramos el certificado SSL para nginx

Eliminamos el archivo **nginx-conf/nginx.conf** actual y lo reemplazamos con el que se encuentra en la carpeta **files/step3/nginx.conf**

~~~
$ rm ./nginx-conf/nginx.conf
$ cp ./files/step3/nginx.conf ./nginx-conf/
~~~

**Importante**, reemplazar en los archivos **nginx.conf** y **docker-compose.yml** la palabra ***dominio*** por el dominio que se usará para el despliegue, y agregar un email valido en el servicio ***certbot*** del archivo **docker-compose.yml**:


#### Configurar puerto HTTPS para Wordpress

***Si usa NginxProxyManager:***

Modificar proxy host creado anteriormente

~~~
Domain Names: dominio www.dominio
Scheme: https
IP: tu-ip
Port: 8443
~~~

***Si no usa NginxProxyManager:***

Habilitar puerto 443 en UFW.

Modificar archivo **docker-compose.yml** y cambiar el puerto de salida del servicio **webserver**:

~~~
...
  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - wordpress-certbot:/etc/letsencrypt
    networks:
      - wordpress-network
...
~~~

#### Reiniciamos el contenedor del servidor web

Ejecutamos el siguiente comando:

~~~
$ docker compose up -d --force-recreate --no-deps webserver
~~~

Revisamos que los contenedores se esten ejecutando:

~~~
$ docker compose ps
~~~

Esperamos una salida semejante a:

~~~
  Name                 Command               State                     Ports
----------------------------------------------------------------------------------------------
certbot     certbot certonly --webroot ...   Exit 0
db          docker-entrypoint.sh --def ...   Up       3306/tcp, 33060/tcp
webserver   nginx -g daemon off;             Up       0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
wordpress   docker-entrypoint.sh php-fpm     Up       9000/tcp
~~~

## Paso 4: Realizar instalación de Wordpress

Ingresamos a la interfaz gráfica de Wordpress, diriguiendonos al nombre de dominio y realizamos la instalación.

~~~
https://dominio
~~~

![Wordpress Install](https://assets.digitalocean.com/articles/docker-wordpress/wp_language_select.png)


## Paso 5: Renovación automática de certificados

Crearemos una nueva carpeta llamada: **scripts**

~~~
$ mkdir scripts
~~~

Agregaremos el script que nos servirá para renovar automáticamente los certificados SSL

~~~
$ cp ./files/extra/ssl-renew.sh ./scripts/
~~~

**Importante**, en el archivo **ssl-renew.sh** modificar la ruta al bin docker y docker compose según corresponda, y la ruta a la carpeta de proyecto también.

Haremos que el script sea ejecutable:

~~~
$ chmod +x ./scripts/ssl-renew.sh
~~~

Creamos una tarea cron para que se ejecute cada cierto tiempo:

~~~
$ sudo crontab -e
~~~

Y agregamos lo siguiente:

~~~
0 12 * * * /root/wordpress-dockercompose/scripts/ssl-renew.sh >> /var/log/cron.log 2>&1
~~~

Podemos configurar el tiempo de ejecución del script según creamos conveniente.

**Importante**, modificar la ruta hacia el script.


## Créditos

Gracias por llegar al final de esta guía.
El artículo original en el que esta basado lo pueden encontrar en:

[DigitalOcean: How To Install WordPress With Docker Compose][OriginalArticle]

![Blog Digital Ocean](https://community-cdn-digitalocean-com.global.ssl.fastly.net/WHDWEF6hZYso7fxQr7ruE3RK)





[OriginalArticle]:https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose#installing-wordpress-with-docker-compose