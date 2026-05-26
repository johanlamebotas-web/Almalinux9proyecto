# IMPLEMENTACIÓN DE SERVIDOR WEB NGINX Y PHP-FPM DESDE CÓDIGO FUENTE

> Proyecto de implementación y configuración de un entorno web utilizando **NGINX 1.31.x** y **PHP 8.4.x** compilados manualmente desde código fuente sobre **AlmaLinux**.

---

# DATOS DEL PROYECTO

| Campo | Información |
|---|---|
| Sistema Operativo | AlmaLinux |
| Servidor Web | NGINX 1.31.x |
| Lenguaje Backend | PHP 8.4.x |
| Método de Instalación | Compilación desde código fuente |
| Comunicación | FastCGI mediante UNIX Socket |
| Ruta de Instalación | `/srv/nginx` |
| Socket UNIX | `/tmp/php84.sock` |

---

# INTEGRANTES

- Daniel Enrique Barrera Ramirez
- Hurtado Galván Johan Manuel
- Hernandez Bautista Angel Gabriel

---

# OBJETIVO GENERAL

Implementar y configurar un stack web compuesto por **NGINX versión 1.31.x** y **PHP-FPM versión 8.4.x** compilados desde código fuente sobre el sistema operativo AlmaLinux, con el propósito de optimizar el rendimiento, la seguridad y el control de configuración del servidor web.

---

# OBJETIVOS ESPECÍFICOS

- Compilar NGINX desde código fuente utilizando el prefijo de instalación en `/srv/nginx`.
- Compilar PHP 8.4 desde código fuente con soporte para:
  - Procesamiento de imágenes
  - Manejo de fechas
  - Internacionalización (`intl`)
- Configurar la comunicación FastCGI mediante sockets UNIX utilizando `/tmp/php84.sock`.
- Implementar servicios administrados por SystemD.
- Habilitar el inicio automático de NGINX y PHP-FPM.
- Ajustar permisos y propietarios para garantizar la comunicación correcta entre ambos servicios.

---

# ARQUITECTURA DEL PROYECTO

```text
Cliente Web
     │
     ▼
┌─────────────┐
│   NGINX     │
│  Puerto 80  │
└──────┬──────┘
       │ FastCGI
       ▼
┌──────────────────┐
│ PHP-FPM 8.4.x    │
│ /tmp/php84.sock  │
└──────────────────┘
```

---

# PREPARACIÓN DEL ENTORNO

## Actualización del sistema

```bash
sudo dnf update -y
```

---

## Instalación de dependencias

```bash
sudo dnf groupinstall "Development Tools" -y
```

```bash
sudo dnf install -y \
gcc \
gcc-c++ \
make \
wget \
tar \
openssl-devel \
pcre-devel \
zlib-devel \
libxml2-devel \
sqlite-devel \
curl-devel \
libjpeg-devel \
libpng-devel \
freetype-devel \
oniguruma-devel \
libicu-devel \
bzip2-devel
```

---

# CREACIÓN DE USUARIOS Y GRUPOS

## Usuario y grupo para NGINX

```bash
sudo groupadd nginx
sudo useradd -r -g nginx -s /sbin/nologin nginx
```

---

## Usuario y grupo para PHP-FPM

```bash
sudo useradd -r -g nginx -s /sbin/nologin php
```

---

# IMPLEMENTACIÓN DE NGINX

## Descarga del código fuente

```bash
cd /usr/local/src
```

```bash
wget https://nginx.org/download/nginx-1.31.0.tar.gz
```

```bash
tar -xvzf nginx-1.31.0.tar.gz
```

```bash
cd nginx-1.31.0
```

---

# CONFIGURACIÓN Y COMPILACIÓN

```bash
./configure \
--prefix=/srv/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module
```

---

## Compilación e instalación

```bash
make
```

```bash
sudo make install
```

---

# VALIDACIÓN DE NGINX

```bash
/srv/nginx/sbin/nginx -v
```

Resultado esperado:

```text
nginx version: nginx/1.31.x
```

---

# IMPLEMENTACIÓN DE PHP 8.4.x

## Descarga del código fuente

```bash
cd /usr/local/src
```

```bash
wget https://www.php.net/distributions/php-8.4.0.tar.gz
```

```bash
tar -xvzf php-8.4.0.tar.gz
```

```bash
cd php-8.4.0
```

---

# CONFIGURACIÓN DE PHP

```bash
./configure \
--prefix=/srv/nginx \
--enable-fpm \
--with-fpm-user=php \
--with-fpm-group=nginx \
--with-zlib \
--with-curl \
--with-openssl \
--enable-mbstring \
--with-jpeg \
--with-freetype \
--enable-intl
```

---

# COMPILACIÓN E INSTALACIÓN DE PHP

```bash
make
```

```bash
sudo make install
```

---

# CONFIGURACIÓN DE PHP-FPM

## Copiar archivos de configuración

```bash
cp php.ini-development /srv/nginx/lib/php.ini
```

```bash
cp /srv/nginx/etc/php-fpm.conf.default /srv/nginx/etc/php-fpm.conf
```

```bash
cp /srv/nginx/etc/php-fpm.d/www.conf.default /srv/nginx/etc/php-fpm.d/www.conf
```

---

# CONFIGURACIÓN DEL SOCKET UNIX

Editar:

```bash
nano /srv/nginx/etc/php-fpm.d/www.conf
```

Modificar:

```ini
user = php
group = nginx

listen = /tmp/php84.sock

listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

---

# CONFIGURACIÓN DE NGINX PARA FASTCGI

Editar:

```bash
nano /srv/nginx/conf/nginx.conf
```

Agregar:

```nginx
server {
    listen 80;
    server_name localhost;

    root /srv/nginx/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;

        fastcgi_pass unix:/tmp/php84.sock;

        fastcgi_index index.php;

        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

# CREACIÓN DE ARCHIVO PHP DE PRUEBA

```bash
nano /srv/nginx/html/phpinfo.php
```

Contenido:

```php
<?php
phpinfo();
?>
```

---

# CONFIGURACIÓN DE SYSTEMD PARA NGINX

Crear:

```bash
nano /etc/systemd/system/nginx.service
```

Contenido:

```ini
[Unit]
Description=NGINX Web Server
After=network.target

[Service]
Type=forking
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/srv/nginx/sbin/nginx -s reload
ExecStop=/srv/nginx/sbin/nginx -s quit
PIDFile=/srv/nginx/logs/nginx.pid

[Install]
WantedBy=multi-user.target
```

---

# CONFIGURACIÓN DE SYSTEMD PARA PHP-FPM

Crear:

```bash
nano /etc/systemd/system/php-fpm8.4.service
```

Contenido:

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
ExecStart=/srv/nginx/sbin/php-fpm
ExecStop=/bin/kill -SIGQUIT $MAINPID

[Install]
WantedBy=multi-user.target
```

---

# RECARGA DE SYSTEMD

```bash
sudo systemctl daemon-reload
```

---

# INICIO DE SERVICIOS

```bash
sudo systemctl start nginx
```

```bash
sudo systemctl start php-fpm8.4
```

---

# HABILITAR AUTOARRANQUE

```bash
sudo systemctl enable nginx
```

```bash
sudo systemctl enable php-fpm8.4
```

---

# VALIDACIÓN DE SERVICIOS

## Verificar estado de NGINX

```bash
systemctl status nginx
```

---

## Verificar estado de PHP-FPM

```bash
systemctl status php-fpm8.4
```

---

# VALIDACIÓN DEL SOCKET UNIX

```bash
ls -l /tmp/php84.sock
```

Resultado esperado:

```text
srw-rw---- 1 nginx nginx
```

---

# PRUEBAS FUNCIONALES

Abrir navegador:

```text
http://localhost/phpinfo.php
```

Resultado esperado:

- Visualización correcta de `phpinfo()`
- Información de módulos PHP
- Confirmación de FastCGI funcionando correctamente

---

# PROBLEMAS ENCONTRADOS

## Error: Permission Denied

Durante las pruebas iniciales se presentó el error:

```text
Permission Denied
```

### Solución aplicada

Se modificaron los permisos del socket UNIX:

```ini
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

Esto permitió que NGINX pudiera acceder correctamente al socket generado por PHP-FPM.

---

# ESTRUCTURA FINAL DEL PROYECTO

```text
/srv/nginx
├── conf
├── html
│   └── phpinfo.php
├── logs
├── sbin
├── lib
└── etc
```

---

# RESULTADOS OBTENIDOS

- Instalación exitosa de NGINX 1.31.x
- Instalación exitosa de PHP 8.4.x
- Comunicación FastCGI funcional
- Socket UNIX operativo
- Servicios administrados por SystemD
- Autoarranque habilitado
- Correcta interpretación de archivos PHP

---

# CONCLUSIONES

La implementación de NGINX y PHP-FPM compilados desde código fuente permitió obtener un mayor control sobre la configuración del entorno web, optimizando tanto el rendimiento como la seguridad del sistema.

El uso de sockets UNIX demostró ser una alternativa eficiente frente al uso de conexiones TCP locales, reduciendo consumo de recursos y mejorando la comunicación entre servicios.

Uno de los principales retos identificados fue la administración de permisos sobre los sockets UNIX, especialmente al enfrentar el error `Permission Denied`.

La solución aplicada mediante la configuración de:

- `listen.owner`
- `listen.group`
- `listen.mode`

permitió comprender la importancia de la correcta administración de permisos dentro de sistemas Linux.

Finalmente, la integración de servicios con SystemD facilitó la automatización y administración completa del entorno.

---

# BIBLIOGRAFÍA

- NGINX Documentation  
  https://nginx.org/en/docs/

- PHP Official Documentation  
  https://www.php.net/docs.php

- AlmaLinux Documentation  
  https://wiki.almalinux.org/

- SystemD Documentation  
  https://www.freedesktop.org/wiki/Software/systemd/

---
