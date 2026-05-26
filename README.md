# Taller_de_Sistemas_Operativos
Institución: Tecnológico de Estudios Superiores de Oriente del Estado de México

Asignatura: Taller de Sistemas Operativos

Proyecto: Implementación e Integración de Servidores NGINX 1.31.x y PHP 8.4.x mediante Compilación Nativa y Sockets Unix

Fecha: 26 de mayo 2026

INTEGRANTES DEL EQUIPO:
1. Hernandez Marmolejo Monica Guadalupe

--------------------------------------------------------------------------------------------------
1. OBJETIVO GENERAL

Instalar, configurar y desplegar un entorno de servidor web compuesto por NGINX (versión 1.31.x) y PHP-FPM (versión 8.4.x), compilados e instalados desde su código fuente en un directorio personalizado (/srv/nginx) sobre la distribución AlmaLinux, asegurando su integración mediante sockets UNIX y su gestión automatizada a través de servicios SystemD.

--------------------------------------------------------------------------------------------------
2. OBJETIVOS ESPECÍFICOS

* Compilar e instalar NGINX 1.31.x y PHP 8.4.x estableciendo /srv/nginx como el prefijo de instalación común en un entorno empresarial compatible con RHEL.
* Configurar los usuarios y grupos de sistema requeridos ('NGINX' y 'PHP:NGINX') aplicando el principio de menor privilegio para asegurar el entorno operativo de AlmaLinux.
* Habilitar extensiones críticas en PHP (procesamiento de imágenes 'gd', manejo de fechas e internacionalización 'intl') resolviendo sus dependencias de desarrollo mediante repositorios nativos y EPEL.
* Crear y registrar los archivos de servicio SystemD para automatizar el arranque coordinado de NGINX y PHP-FPM en el inicio del sistema operativo bajo las directivas graphical.target o multi-user.target.
* Configurar la comunicación interna utilizando el protocolo FastCGI a través de un Socket UNIX local (/tmp/php84.sock) para optimizar el rendimiento y eliminar el overhead de la pila de red local TCP/IP.
* Validar el correcto funcionamiento integral del entorno web mediante el despliegue de un script de prueba de diagnóstico ('phpinfo.php').

--------------------------------------------------------------------------------------------------
3. DESARROLLO DEL PROYECTO


3.1: Preparación del Sistema y Dependencias de Compilación
En la distribución AlmaLinux, es necesario activar el grupo de herramientas de desarrollo y los repositorios EPEL para poder resolver de forma nativa las bibliotecas de desarrollo requeridas:

Comandos de consola:

$ sudo dnf install -y epel-release

$ sudo dnf groupinstall -y "Development Tools"

$ sudo dnf install -y pcre-devel zlib-devel openssl-devel libxml2-devel sqlite-devel libcurl-devel libpng-devel libjpeg-turbo-devel libwebp-devel freetype-devel oniguruma-devel libicu-devel pkgconfig

3.2: Configuración de Usuarios y Grupos de Sistema
Para mitigar la superficie de ataque y asegurar los privilegios de los procesos demonio en el sistema operativo:

# Crear grupo y usuario del sistema para NGINX
$ sudo groupadd --system NGINX

$ sudo useradd -s /sbin/nologin --system -g NGINX NGINX

# Crear usuario para PHP integrado al grupo de NGINX
$ sudo useradd -s /sbin/nologin --system -g NGINX PHP

3.3: Descarga, Compilación e Instalación de NGINX 1.31.x
1. Obtención y extracción del código fuente de la rama de desarrollo estable:
   
$ wget https://nginx.org/download/nginx-1.31.2.tar.gz

$ tar -zxvf nginx-1.31.2.tar.gz

$ cd nginx-1.31.2

3. Configuración de parámetros de compilación con prefijo modular y asignación de propietario:
$ ./configure --prefix=/srv/nginx --user=NGINX --group=NGINX --with-http_ssl_module

4. Ejecución de la construcción e instalación nativa:
$ make
$ sudo make install

3.4: Descarga, Compilación e Instalación de PHP 8.4.x
1. Obtención del código fuente oficial del intérprete PHP:
   
$ wget https://www.php.net/distributions/php-8.4.1.tar.gz

$ tar -zxvf php-8.4.1.tar.gz

$ cd php-8.4.1

3. Script de configuración habilitando PHP-FPM, aislamiento de usuarios y los módulos de internacionalización (intl), procesamiento gráfico (external-gd), multibyte (mbstring) y manejo de calendarios:
$ ./configure --prefix=/srv/nginx --with-config-file-path=/srv/nginx/html --enable-fpm --with-fpm-user=PHP --with-fpm-group=NGINX --enable-intl --with-external-gd --enable-mbstring --with-openssl --with-curl --with-zlib --enable-calendar

4. Compilación e instalación en el prefijo unificado:
   
$ make

$ sudo make install

6. Despliegue del archivo de configuración para producción:
   
$ sudo cp php.ini-production /srv/nginx/lib/php.ini

3.5: Configuración de PHP-FPM y del Intercambio vía Socket UNIX

Se replican las plantillas por defecto en el directorio de producción:

$ sudo cp /srv/nginx/etc/php-fpm.conf.default /srv/nginx/etc/php-fpm.conf

$ sudo cp /srv/nginx/etc/php-fpm.d/www.conf.default /srv/nginx/etc/php-fpm.d/www.conf

Modificamos el archivo '/srv/nginx/etc/php-fpm.d/www.conf' para sustituir la escucha de red TCP por el socket UNIX local con los permisos restringidos del grupo:

Contenido de /srv/nginx/etc/php-fpm.d/www.conf:
--------------------------------------------------
[www]
user = PHP
group = NGINX

listen = /tmp/php84.sock
listen.owner = PHP
listen.group = NGINX
listen.mode = 0660

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
--------------------------------------------------

3.6: Configuración de NGINX para la Comunicación FastCGI
Se edita el archivo maestro '/srv/nginx/conf/nginx.conf' para enrutar el procesamiento dinámico PHP directamente al socket creado:

Contenido de /srv/nginx/conf/nginx.conf:
--------------------------------------------------
user NGINX NGINX;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;
        root         /srv/nginx/html;
        index        index.php index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }

        # Integración con PHP-FPM vía Socket UNIX local
        location ~ \.php$ {
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_pass   unix:/tmp/php84.sock;
        }
    }
}
--------------------------------------------------

3.7: Automatización con Unidades de Servicio SystemD
Para asegurar que los servicios web inicien de manera persistente con el sistema operativo, se registran las unidades correspondientes.

Unidad para NGINX (/etc/systemd/system/nginx.service)
--------------------------------------------------
[Unit]

Description=The NGINX HTTP and reverse proxy server

After=network.target remote-fs.target nss-lookup.target

[Service]

Type=forking

PIDFile=/srv/nginx/logs/nginx.pid

ExecStartPre=/srv/nginx/sbin/nginx -t

ExecStart=/srv/nginx/sbin/nginx

ExecReload=/srv/nginx/sbin/nginx -s reload

ExecStop=/bin/kill -s QUIT $MAINPID

PrivateTmp=true

[Install]

WantedBy=multi.user.target graphical.target

Unidad para PHP-FPM (/etc/systemd/system/php-fpm8.4.service)
--------------------------------------------------
[Unit]

Description=The PHP 8.4 FastCGI Process Manager

After=network.target

[Service]

Type=simple

PIDFile=/srv/nginx/var/run/php-fpm.pid

ExecStart=/srv/nginx/sbin/php-fpm --nodaemonize --fpm-config /srv/nginx/etc/php-fpm.conf

ExecReload=/bin/kill -USR2 $MAINPID

PrivateTmp=false

[Install]

WantedBy=multi-user.target graphical.target


#Habilitación y Arranque en Consola:
--------------------------------------------------

$ sudo systemctl daemon-reload

$ sudo systemctl enable nginx.service --now

$ sudo systemctl enable php-fpm8.4.service --now

3.8: Comprobación y Validación del Funcionamiento
Para validar de forma definitiva la integración del stack web, se crea el script dinámico de pruebas en la raíz del servidor:

$ echo "<?php phpinfo(); ?>" | sudo tee /srv/nginx/html/phpinfo.php
$ sudo chmod 644 /srv/nginx/html/phpinfo.php

Al acceder desde un navegador web a 'http://localhost/phpinfo.php', se confirmará que la API del servidor reporta 'FPM/FastCGI', se lee correctamente el socket unix '/tmp/php84.sock' y se encuentran cargados los módulos core solicitados ('gd', 'date', 'intl').

--------------------------------------------------------------------------------------------------
4. CONCLUSIONES

* La compilación desde el código fuente en sistemas de nivel empresarial como AlmaLinux proporciona un entorno optimizado y minimiza la superficie de ataque al incluir únicamente los módulos estrictamente necesarios para la operación del negocio.
* El uso de Sockets UNIX en lugar de Sockets TCP (127.0.0.1:9000) reduce el overhead de la pila de red local, eliminando la necesidad de manejar cabeceras de red internas y acelerando drásticamente el procesamiento de peticiones FastCGI entre NGINX y PHP-FPM.
* El aislamiento por usuarios (NGINX y PHP) garantiza que, en caso de que una vulnerabilidad comprometa el intérprete de PHP, el atacante no obtenga control automático sobre el binario o las configuraciones críticas del servidor web NGINX.
* La correcta estructuración de las unidades de servicio SystemD vinculadas a los targets 'multi-user' y 'graphical' dota a la infraestructura de resiliencia operativa ante fallos catastróficos o reinicios programados del nodo.

--------------------------------------------------------------------------------------------------
5. BIBLIOGRAFÍA

* International Components for Unicode. (2025). ICU User Guide. https://icu.unicode.org/
* NGINX Documentation. (2026). NGINX Core functionality. https://nginx.org/en/docs/
* PHP Documentation Group. (2026). FastCGI Process Manager (FPM). https://www.php.net/manual/en/install.fpm.php
* The SystemD Consortium. (2025). systemd.service - Service unit configuration. https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
