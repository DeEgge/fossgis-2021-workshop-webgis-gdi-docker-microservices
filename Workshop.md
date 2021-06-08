# FOSSGIS 2021 Workshop WebGIS-GDI mit Docker Microservices

**2021-06-09**

![WebGIS-GDI mit Docker Microservices](webgis_dockerisieren.png)

## Inhalt

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [FOSSGIS 2021 Workshop WebGIS-GDI mit Docker Microservices](#fossgis-2021-workshop-webgis-gdi-mit-docker-microservices)
  - [Inhalt](#inhalt)
  - [Christian Kuntzsch](#christian-kuntzsch)
- [Workshop](#workshop)
  - [1. Einleitung](#1-einleitung)
  - [3. Apache + php-fpm](#3-apache--php-fpm)
  - [4. Apache durch nginx austauschen](#4-apache-durch-nginx-austauschen)
    - [4.1 Persistente Daten in Container einbinden](#41-persistente-daten-in-container-einbinden)
  - [5. Container für Postgres/PostGIS-DB](#5-container-für-postgrespostgis-db)
  - [6. Mapserver](#6-mapserver)
  - [7. Redis Session Handler](#7-redis-session-handler)
  - [Finale `docker-compose.yml`](#finale-docker-composeyml)

<!-- /code_chunk_output -->

---

## Christian Kuntzsch

* 💼 [WhereGroup GmbH](https://wheregroup.com)
* 📧 christian.kuntzsch@wheregroup.com
* 🐦 [@DeEgge](https://twitter.com/DeEgge)
* 👨‍💻 Software-Entwickler seit 2017, zuletzt starker Fokus auf Containerisierung mit Docker und Deployment in Kubernetes

# Workshop

## 1. Einleitung

* Docker? Microservices? Docker Microservices?
* Was wir machen wollen...
* [Docker Cheat Sheet](https://dockerlabs.collabnix.com/docker/cheatsheet/)
* [Docker-Compose Cheat Sheet](https://devhints.io/docker-compose)

<!-- ### 1.1 Vorbereitung

Aktuelles Mapbender-Release herunterladen

```shell
mkdir -p ~/workshop
wget https://mapbender.org/builds/mapbender-starter-current.tar.gz -O ~/workshop/mapbender-starter-current.tar.gz
tar -zxf ~/workshop/mapbender-starter-current.tar.gz -C ~/workshop
mv $(ls -d ~/workshop/*/ | grep mapbender) ~/workshop/mapbender/
``` -->

## 2. Apache + mod_php

* simpelster Anwendungsfall, kein `docker-compose`

Dockerfile für **Apache + mod_php** anlegen

```shell
cd ~/workshop
mkdir -p apache-mod-php
code . # oder Ähnliches...
```

[Welche Bibliotheken/PHP-Module/... müssen für Mapbender installiert werden?](https://doc.mapbender.org/de/installation/installation_ubuntu.html#vorbereitung)

<details>
<summary>

`apache-mod-php/Dockerfile`

</summary>

```Docker
FROM php:7.3-apache

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update \
	&& apt-get install --no-install-recommends -y \
        wget \
		libzip-dev \
		libbz2-dev \
		libwebp-dev \
		libjpeg-dev \
		libpng-dev \
		libicu-dev \
	&& rm /etc/apache2/sites-enabled/* \
	&& docker-php-ext-install zip \
	&& docker-php-ext-install bz2 \
	&& docker-php-ext-install gd \
	&& docker-php-ext-install intl \
	&& a2enmod rewrite

RUN wget https://mapbender.org/builds/mapbender-starter-current.tar.gz -O /tmp/mapbender-starter-current.tar.gz \
    && tar -zxf /tmp/mapbender-starter-current.tar.gz -C /tmp \
    && mv $(ls -d /tmp/*/ | grep mapbender) /var/www/mapbender/ \
    && chown -R www-data:www-data /var/www/mapbender

COPY ./mapbender.conf /etc/apache2/sites-enabled/mapbender.conf

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

EXPOSE 80
```

</details>

<details>

<summary>

`apache-mod-php/mapbender.conf` [Quelle](https://doc.mapbender.org/de/installation/installation_ubuntu.html#konfiguration-apache-2-4)

</summary>

```
Alias / /var/www/mapbender/web/

 <Directory /var/www/mapbender/web/>
    Options MultiViews FollowSymLinks

    AllowOverride All
    DirectoryIndex app.php
    Require all granted

    RewriteEngine On
    RewriteBase /
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ app.php [QSA,L]
</Directory>
```

</details>

```shell
# Docker Container bauen
cd apache-mod-php
docker build -t mapbender_apache-mod-php .

# Docker Container starten
docker run -d --name mapbender_apache-mod-php -p 8081:80 mapbender_apache-mod-php:latest

# Ggfs. User- und Group-IDs an Host-User anpassen
docker exec -t -e USER_ID=$(id -u) -e GROUP_ID=$(id -g) mapbender_apache-mod-php /bin/bash -c "usermod -u $USER_ID www-data && groupmod -g $GROUP_ID www-data"
```

Aktuelle Ordnerstruktur

```
📦workshop
 ┣ 📂apache-mod-php
 ┃ ┣ 📜Dockerfile
 ┃ ┗ 📜mapbender.conf
```

## 3. Apache + php-fpm

Trennen von statischem Webserver (Apache) und PHP-Prozessierung (php-fpm), mittels `docker-compose`

```shell
cd ~/workshop
mkdir apache php
```

<details>
<summary>

`apache/Dockerfile`

</summary>

```Docker
FROM debian:buster-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        apache2 \
    && rm /etc/apache2/sites-enabled/*
RUN a2enmod proxy_fcgi rewrite proxy proxy_balancer proxy_http proxy_ajp
RUN sed -i '/Global configuration/a \
    ServerName localhost \
    ' /etc/apache2/apache2.conf
EXPOSE 80
CMD apachectl -DFOREGROUND -e info
```

</details>

<details>
<summary>

`php/Dockerfile`

</summary>

```Docker
FROM php:7.3-fpm

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update \
	&& apt-get install --no-install-recommends -y \
		wget \
		libzip-dev \
		libbz2-dev \
		libwebp-dev \
		libjpeg-dev \
		libpng-dev \
		libicu-dev \
	&& docker-php-ext-install zip \
	&& docker-php-ext-install bz2 \
	&& docker-php-ext-install gd \
	&& docker-php-ext-install intl \
	&& docker-php-ext-install opcache \
	&& docker-php-ext-enable opcache

ENV PHP_MEMORY_LIMIT=-1

EXPOSE 9000
```

</details>

<details>
<summary>

`apache/mapbender.conf` Ergänzt um proxy-Handler für PHP-FPM

</summary>

```
Alias / /var/www/mapbender/web/

<FilesMatch \.php$>
    SetHandler proxy:fcgi://php:9000
    # for Unix sockets, Apache 2.4.10 or higher
    # SetHandler proxy:unix:/path/to/fpm.sock|fcgi://dummy
</FilesMatch>

 <Directory /var/www/mapbender/web/>
    Options MultiViews FollowSymLinks

    AllowOverride All
    DirectoryIndex app.php
    Require all granted

    RewriteEngine On
    RewriteBase /
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ app.php [QSA,L]
</Directory>
```

</details>

Wir benötigen ein simples Skript, dass Mapbender herunterlädt. Den Inhalt des Verzeichnisses `/var/www/` binden wir als Volume ein, sodass beim Aufruf des Skripts der Ordner befüllt wird.

```shell
# Ordner für persistente Daten anlegen
mkdir ~/workshop/mapbender_shared
```

<details>
<summary>

`mapbender_shared/mapbender-setup.sh`

</summary>

```bash
#!/bin/bash
set -x

echo "Start Mapbender setup"

MAPBENDER_FOLDER=/var/www/mapbender

if [ -n "$USER_UID" ]; then
  # NOT FOR PRODUCTION
  usermod -u $USER_UID www-data
  groupmod -g $USER_GID www-data
fi

echo $APACHE_RUN_DIR

if [ -d "$MAPBENDER_FOLDER/web" ]; then
  echo "$MAPBENDER_FOLDER/web is a directory."
else

echo "$MAPBENDER_FOLDER/web is not a directory."

echo "Downloading latest Mapbender release"
wget https://mapbender.org/builds/mapbender-starter-current.tar.gz -O /var/www/mapbender-starter-current.tar.gz

echo "Extracting..."
tar -zxf /var/www/mapbender-starter-current.tar.gz -C /var/www

echo "Installing into $MAPBENDER_FOLDER"
mv $(ls -d /var/www/*/ | grep mapbender) $MAPBENDER_FOLDER/
chown -R www-data:www-data $MAPBENDER_FOLDER

fi

ls -al "$MAPBENDER_FOLDER"
```

</details>

<details>
<summary>

`docker-compose.yml`

</summary>

```yml
version: "3.1"
services:
  php:
    build: ./php
    volumes:
      - ./volumes/www/:/var/www
      - ./mapbender_shared/mapbender-setup.sh:/tmp/mapbender-setup:ro
    command: bash -c "/tmp/mapbender-setup && docker-php-entrypoint php-fpm"
  apache:
    build: ./apache
    ports:
      - "8081:80"
    volumes:
      - ./volumes/www/:/var/www:ro
      - ./apache/mapbender.conf:/etc/apache2/sites-enabled/mapbender.conf
```

</details>

```shell
# Skript ausführbar machen
chmod +x ~/workshop/mapbender_shared/mapbender-setup.sh

# Container bauen
docker-compose build

# Container starten
docker-compose up

# Beobachtet den Konsolen-Output! Das Setup-Skript installiert Mapbender...
```

```
📦workshop
 ┣ 📂apache
 ┃ ┣ 📜Dockerfile
 ┃ ┗ 📜mapbender.conf
 ┣ 📂mapbender_shared
 ┃ ┗ 📜mapbender-setup.sh
 ┣ 📂php
 ┃ ┗ 📜Dockerfile
 ┣ 📂volumes
 ┃ ┣ mapbender
 ┃ ┃ ┗ ... 
 ┗ 📜docker-compose.yml
```

## 4. Apache durch nginx austauschen

Leichtes Ersetzen von Services (Microservice-Architektur) möglich.

`php-fpm` Image und Container bleiben gleich. Für nginx muss nicht einmal ein Dockerfile geschrieben werden (wenn `default.conf` per Volume überschrieben wird).

<details>
<summary>

`docker-compose.yml`

</summary>

```yml
version: "3.1"
services:
  php:
    build: ./php
    volumes:
      - ./volumes/www/:/var/www
      - ./mapbender_shared/mapbender-setup.sh:/tmp/mapbender-setup:ro
    command: bash -c "/tmp/mapbender-setup && docker-php-entrypoint php-fpm"
  nginx:
    image: nginx:latest
    ports:
      - "8081:80"
    volumes:
      - ./volumes/www/:/var/www
      - ./nginx/mapbender.conf:/etc/nginx/conf.d/default.conf
```

</details>

<details>
<summary>

`nginx/mapbender.conf` [Quelle](https://symfony.com/doc/3.4/setup/web_server_configuration.html#nginx)

</summary>

```
server {
    server_name localhost;
    
    root /var/www/mapbender/web;

    location / {
        gzip_static on;
        try_files $uri /app.php$is_args$args;
    }

    #DEV
    location ~ ^/(app_dev|config)\.php(/|$) {
        gzip on;
        fastcgi_pass php:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
    }

    #PROD
    location ~ ^/app\.php(/|$) {
        gzip on;
        fastcgi_pass php:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        internal;
    }
    
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
}
```

</details>

```shell
# Apache-Container stoppen
docker-compose stop apache
docker-compose up -d nginx
```

```
📦workshop
 ┣ ❌#apache
 ┃ ┣ ❌#Dockerfile
 ┃ ┗ ❌#mapbender.conf
 ┣ 📂mapbender_shared
 ┃ ┣ 📜mapbender-setup.sh
 ┣ 📂nginx
 ┃ ┗ 📜mapbender.conf
 ┣ 📂php
 ┃ ┗ 📜Dockerfile
 ┗ 📜docker-compose.yml
```

### 4.1 Persistente Daten in Container einbinden

Per Volume

<details>
<summary>

`mapbender_shared/FOSSGIS_Demo.yml`

</summary>

```yml
parameters:
    applications:
        fossgis_demo:
            title: FOSSGIS Demo Map
            screenshot: screenshot.png
            description: Fullscreen style, Simple map showing WMS use.
            published: true
            template:  Mapbender\CoreBundle\Template\Fullscreen
            regionProperties:
              - name: sidepane
                properties:
                  name: accordion
            layersets:
                main:
                    WMS BGDI:
                        class: Mapbender\WmsBundle\Entity\WmsInstance
                        url: https://wms.geo.admin.ch/?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetCapabilities
                        version: 1.3.0
                        title: WMS BGDI
                        layers:
                            4: { name: ch.vbs.kataster-belasteter-standorte-militaer,           title : Belastete Standorte Militär,            visible: true }
                            5: { name: ch.vbs.kataster-belasteter-standorte-militaer_polygon,   title : Belastete Standorte Militär Polygon,    visible: true }
                            6: { name: ch.vbs.kataster-belasteter-standorte-militaer_point,     title : Belastete Standorte Militär Punkt,      visible: true }
                            2: { name: ch.are.erreichbarkeit-oev,                               title : Erreichbarkeit mit dem ÖV,              visible: false }
                            3: { name: ch.bafu.laerm-bahnlaerm_nacht,                           title : Eisenbahnlärm Nacht,                    visible: true }
                            1: { name: ch.swisstopo.landeskarte-farbe-10,                       title : Landeskarte 1:10.000 (farbig),          visible: true }
                        info_format: text/xml
                        visible: true
                        format: image/png
                        transparent: true
                        tiled: true
                        opacity: 100
                    osm:
                        class: Mapbender\WmsBundle\Entity\WmsInstance
                        title: OSM Demo
                        url: https://osm-demo.wheregroup.com/service
                        version: 1.3.0
                        layers:
                            30: { name: osm,    title : OSM Demo WhereGroup,  visible: true }
                        info_format: text/html
                        visible: true
                        format: image/png
                        transparent: true
                        tiled: true
                        opacity: 100
                overview:
                    osm_overview:
                        class: Mapbender\WmsBundle\Entity\WmsInstance
                        title: OSM Demo
                        url: https://osm-demo.wheregroup.com/service
                        version: 1.3.0
                        layers:
                            40: { name: osm,    title : OSM Demo WhereGroup,  visible: true }
                        tiled: false
                        format: image/png
                        transparent: true
                        visible: true
                        opacity: 100
            elements:
                toolbar:
                    legend-button:
                        title: mb.core.legend.class.title
                        tooltip: mb.core.legend.class.title
                        class: Mapbender\CoreBundle\Element\Button
                        label: false
                        target: legend
                        icon: iconLegend
                content:
                    map:
                        class: Mapbender\CoreBundle\Element\Map
                        layersets: [main]
                        srs: EPSG:3857
                        extents:
                            start: [974196,5975480,991648,5982547]
                            max: [-500000,4350000,1600000,6850000]
                        scales: [5000000,1000000,500000,100000,50000,25000,10000,7500,5000,2500,1000,500]
                        otherSrs: ["EPSG:25833","EPSG:31466","EPSG:31467","EPSG:3857","EPSG:4326"]
                    zoombar:
                        class: Mapbender\CoreBundle\Element\ZoomBar
                        target: map
                        anchor: right-top
                        draggable: false
                    legend:
                        class: Mapbender\CoreBundle\Element\Legend
                        target: map
                        elementType: dialog
                        autoOpen: false
                        showLayerTitle: true
                        showGroupedLayerTitle: true
                    scalebar:
                        class: Mapbender\CoreBundle\Element\ScaleBar
                        target: map
                        anchor: 'left-bottom'
                        maxWidth: 200
                        units: km
                    overview:
                        class: Mapbender\CoreBundle\Element\Overview
                        target: map
                        layerset: overview
                        width: 200
                        height: 100
                        anchor: 'right-bottom'
                        maximized: true
                        fixed: false
                    scaledisplay:
                        class: Mapbender\CoreBundle\Element\ScaleDisplay
                        target: map
                        anchor: right-top
                        scalePrefix: Scale
                        unitPrefix: true
                sidepane:
                    layertree:
                        class: Mapbender\CoreBundle\Element\Layertree
                        target: map
                        type: element
                        autoOpen: false
                        showBaseSource: true
                        menu: [opacity,zoomtolayer,metadata]
                    sketch:
                        class: Mapbender\CoreBundle\Element\Sketch
                        target: map
                        auto_activate: false
                    coordinatesutility:
                        class: Mapbender\CoordinatesUtilityBundle\Element\CoordinatesUtility
                        target: map
                        srsList: []
                        addMapSrsList: true
                        zoomlevel: 6
                    html-about-mapbender:
                        title: 'About Mapbender'
                        class: Mapbender\CoreBundle\Element\HTMLElement
                        classes: html-element-inline
                        content: "<p>Mapbender is an OSGeo Project. Find out more about OSGeo at<br/><a href=\"https://www.osgeo.org\" target=\"_blank\">https://www.osgeo.org</a><br/><a href=\"https://www.osgeo.org/projects/mapbender/\" target=\"_blank\">https://www.osgeo.org/projects/mapbender/</a></p>"

                footer:
                    activityindicator:
                        class: Mapbender\CoreBundle\Element\ActivityIndicator
                    coordinates:
                        class: Mapbender\CoreBundle\Element\CoordinatesDisplay
                        target: map
                        label: false
                        empty:  "x: - y: -"
                        prefix: "x: "
                        separator:  " y: "
                    srs:
                        class: Mapbender\CoreBundle\Element\SrsSelector
                        tooltip: mb.core.srsselector.class.description
                        label: false
                        target: map
                    scaleselector:
                        class: Mapbender\CoreBundle\Element\ScaleSelector
                        tooltip: mb.core.scaleselector.class.description
                        label: false
                        target: map
                    copyrightButton:
                        title: © OpenStreetMap contributors
                        tooltip: © OpenStreetMap contributors
                        class: Mapbender\CoreBundle\Element\Button
                        label: true
                        click: https://www.openstreetmap.org/copyright
                    HTML:
                        title: HTML-powered by Mapbender
                        class: Mapbender\CoreBundle\Element\HTMLElement
                        content: '<span style="color: #6FB536; font-weight:bold">powered by Mapbender</span>'
                        classes: html-element-inline
```

</details>

<details>
<summary>

`docker-compose.yml`

</summary>

```yml
...
php:
    build: ./php-fpm
    volumes:
      - ./volumes/www/:/var/www
      - ./mapbender_shared/mapbender-setup.sh:/tmp/mapbender-setup:ro
      - ./mapbender_shared/FOSSGIS_Demo.yml:/var/www/mapbender/app/config/applications/FOSSGIS_Demo.yml
      - ./mapebder_shared/FOSSGIS_LOGO_2021.png:/var/www/mapbender/web/uploads/FOSSGIS_Demo_yml/screenshot.png
...
```

</details>

```shell
# Container im Detached-Mode laufen lassen
docker-compose up -d

# Befehl im Service php ausführen
docker-compose exec --user www-data php php /var/www/mapbender/app/console cache:clear --env=prod 
```

```
📦workshop
 ┣ 📂mapbender_shared
 ┃ ┣ 📜FOSSGIS_Demo.yml
 ┃ ┣ 📜FOSSGIS_Logo_2021.png
 ┃ ┣ 📜mapbender-setup.sh
 ┣ 📂nginx
 ┃ ┗ 📜mapbender.conf
 ┣ 📂php
 ┃ ┗ 📜Dockerfile
 ┣ 📂volumes
 ┃ ┣ mapbender
 ┃ ┃ ┗ ... 
 ┗ 📜docker-compose.yml
```

## 5. Container für Postgres/PostGIS-DB

Kann ebenfalls ohne separates Dockerfile direkt als Image eingebunden werden.
Datenverzeichnis wird im Host eingebunden, sodass ein Datenbankupdate leicht möglich ist.

PHP-Image aktualisieren und `pgsql`-Treiber für PHP installieren

`php/Dockerfile`

```Docker
...
RUN apt-get install --no-install-recommends -y \
 		libpq-dev \
 	&& docker-php-ext-install pdo_pgsql
...
```

```shell
# PHP-Container stoppen
docker-compose stop php

# PHP-Container neu bauen
docker-compose build php

# PHP-Container wieder starten
docker-compose up -d php
```

Datenbank-Parameter für Mapbender mitgeben, in Form von `parameters.yml`

<details>
<summary>

`mapbender_shared/parameters.yml`

</summary>

```yml
parameters:
### todo: switch to using dsn urls, see https://www.doctrine-project.org/projects/doctrine-dbal/en/2.9/reference/configuration.html#connecting-using-a-url
###       To avoid suprises, a DSN can be constructed here from parts, at least for the shipping default parameters
###       Connection definition in config.yml should only use url: %database_dsn% though
    database_driver:   '%env(DATABASE_DRIVER)%'
    database_host:     '%env(DATABASE_HOST)%'
    database_port:     '%env(DATABASE_PORT)%'
    database_name:     '%env(DATABASE_NAME)%'
    database_path:     '%env(DATABASE_PATH)%'
    database_user:     '%env(DATABASE_USER)%'
    database_password: '%env(DATABASE_PASSWORD)%'

    mailer_transport:  smtp
    mailer_host:       localhost
    mailer_user:       ~
    mailer_password:   ~

    # locale en, de, it, es, ru, nl, pt are available
    fallback_locale:   en
    locale:            en
    secret:            ThisTokenIsNotSoSecretChangeIt

    ## Legacy branding / versioning params.
    ## This is no longer used for versioning Mapbender and will never be updated
    ## again for a Mapbender release.
    ## For BC / continuity, you may still use these variables to brand / version your project.
    ## For summary information / discussion see https://github.com/mapbender/mapbender/pull/1012
    ## For full project branding / versioning options, see code comment:
    ## https://github.com/mapbender/mapbender/blob/42e0f8b9a8031118719fc4881a92f0adab4ebacf/src/Mapbender/CoreBundle/DependencyInjection/Compiler/ProvideBrandingPass.php#L17
    fom: ~

    # framework : http://symfony.com/doc/2.8/reference/configuration/framework.html#cookie-lifetime
    cookie_secure: false
    cookie_lifetime: 3600

# OWSProxy Configuration
# see https://github.com/mapbender/owsproxy3/blob/master/CONFIGURATION.md#extension-configuration
    ows_proxy3_logging: false
    ows_proxy3_obfuscate_client_ip: true
    ows_proxy3_host: ~
    ows_proxy3_port: ~
    ows_proxy3_connecttimeout: 60
    ows_proxy3_timeout: 90
    ows_proxy3_user: ~
    ows_proxy3_password: ~
    ows_proxy3_noproxy: ~

    # default layer order when creating *new* WMS layerset instances
    # allowed values are either
    # * "standard": Traditional Mapbender behaviour: top-down rendering in GetCapabilities order;
    #               also the default if this parameter is not defined
    # * "reverse": bottom-up, for QGIS server, ArcGIS etc
    #     wms.default_layer_order: standard

```

</details>

<details>
<summary>

`docker-compose.yml`

</summary>

```yml
services:
  ...
  php:
    volumes:
    ...
      - ./mapbender_shared/parameters.yml:/var/www/mapbender/app/config/parameters.yml
    environment:
      DATABASE_DRIVER: pdo_pgsql
      DATABASE_HOST: postgis
      DATABASE_PORT: 5432
      DATABASE_PATH: "~"
      DATABASE_NAME: mapbender
      DATABASE_USER: mb_db_user
      DATABASE_PASSWORD: mb_db_pass
  ...
  postgis:
    image: postgis/postgis:12-3.1-alpine
    environment:
      POSTGRES_DB: mapbender
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres_pass
    volumes:
      - ./volumes/postgresql/:/var/lib/postgresql/data
```

</details>

```shell
# PostgreSQL starten
docker-compose up -d postgis

# Ggfs. User-ID und Group-ID an Hostuser anpassen
docker-compose exec -e USER_ID=$(id -u) -e GROUP_ID=$(id -g) postgis /bin/bash -c "usermod -u $USER_ID postgres && groupmod -g $GROUP_ID postgres"

# psql im Container ausführen
docker-compose exec postgis su -s /bin/bash postgres
```

```sql
-- Anwendungsuser erzeugen
CREATE USER mb_db_user WITH ENCRYPTED PASSWORD 'mb_db_pass';

-- Datenbank erzeugen
CREATE DATABASE mapbender; -- not needed when ENV POSTGRES_DB=mapbender

-- grants
GRANT ALL PRIVILEGES ON DATABASE mapbender TO mb_db_user;
```

Mapbender-DB einrichten [Quelle](https://doc.mapbender.org/de/installation/installation_ubuntu.html#optional)

```shell
docker-compose exec --user www-data php php /var/www/mapbender/app/console doctrine:schema:create
docker-compose exec --user www-data php php /var/www/mapbender/app/console mapbender:database:init -v
docker-compose exec --user www-data php php /var/www/mapbender/app/console fom:user:resetroot
docker-compose exec --user www-data php php /var/www/mapbender/app/console cache:clear --env=prod
```

**Optional**

Datenbankinitialisierung in Skript auslagern

https://github.com/docker-library/docs/tree/master/postgres#initialization-scripts

```
📦workshop
 ┣ 📂mapbender_shared
 ┃ ┣ 📜FOSSGIS_Demo.yml
 ┃ ┣ 📜FOSSGIS_Logo_2021.png
 ┃ ┣ 📜mapbender-setup.sh
 ┃ ┗ 📜parameters.yml
 ┣ 📂nginx
 ┃ ┗ 📜mapbender.conf
 ┣ 📂php
 ┃ ┗ 📜Dockerfile
 ┣ 📂volumes
 ┃ ┣ 📂postgresql
 ┃ ┃ ┗ ... 
 ┃ ┣ mapbender
 ┃ ┃ ┗ ... 
 ┗ 📜docker-compose.yml
```

## 6. Mapserver

Empfehlung für Docker-Image: [`pdok/mapserver`](https://hub.docker.com/r/pdok/mapserver)

  * Verbirgt Mapfile-String beim Request
  * Nutzt lighttpd anstatt Apache

<details>
<summary>

Mapfile `mapserver/data/bar_cafe.map`

</summary>

```
MAP
  NAME          ""
  CONFIG        "MS_ERRORFILE" "stderr"
  EXTENT        7 46 10 48
  UNITS         meters
  STATUS        ON
  SIZE          5000 5000
  
  ## global debug settings for mapserver, remove comment in lines below to enable
  # DEBUG         4         # https://mapserver.org/optimization/debugging.html
  # CONFIG "CPL_DEBUG" "ON" # GDAL

  RESOLUTION 91
  DEFRESOLUTION 91

  SYMBOL
    TYPE ellipse
    POINTS 1 1 END
    NAME "circle"
  END

  PROJECTION
    "init=epsg:4326"
  END

  WEB
    METADATA
      "ows_enable_request"               "*"
      "ows_fees"                         "NONE"
      "ows_contactorganization"          "Unknown"
      "ows_schemas_location"             "http://schemas.opengis.net"
      "ows_service_onlineresource"       "http://localhost"
      "ows_contactperson"                "ContactCenter Unknown"
      "ows_contactposition"              "pointOfContact"
      "ows_contactvoicetelephone"        ""
      "ows_contactfacsimiletelephone"    ""
      "ows_addresstype"                  ""
      "ows_address"                      ""
      "ows_city"                         "City"
      "ows_stateorprovince"              ""
      "ows_postcode"                     ""
      "ows_country"                      "Country"
      "ows_contactelectronicmailaddress" "example@unknown.org"
      "ows_hoursofservice"               ""
      "ows_contactinstructions"          ""
      "ows_role"                         ""
      "ows_srs"                          "EPSG:4326 EPSG:3857 EPSG:4258 EPSG:900913 CRS:84"
      "ows_accessconstraints"            "otherRestrictions;http://creativecommons.org/publicdomain/mark/1.0"      
    END
  END

 # outputformat used by WMS GetFeatureInfo and the WFS GetFeature requests
  OUTPUTFORMAT
    NAME "GEOJSON"       # format name (visible as format in the 1.0.0 capabilities)
    DRIVER "OGR/GEOJSON"
    MIMETYPE "application/json; subtype=geojson"
    FORMATOPTION "STORAGE=stream"
    FORMATOPTION "FORM=SIMPLE"
    FORMATOPTION "USE_FEATUREID=true"
    FORMATOPTION "LCO:ID_FIELD=fid"
    FORMATOPTION "LCO:ID_TYPE=STRING"
  END

  # outputformat used by WMS GetFeatureInfo and the WFS GetFeature requests
  OUTPUTFORMAT
    NAME "JSON"
    DRIVER "OGR/GEOJSON"
    MIMETYPE "application/json"
    FORMATOPTION "STORAGE=stream"
    FORMATOPTION "FORM=SIMPLE"
    FORMATOPTION "USE_FEATUREID=true"
    FORMATOPTION "LCO:ID_FIELD=fid"
    FORMATOPTION "LCO:ID_TYPE=STRING"
  END
   
  # outputformat used by WMS GetMap requests
  OUTPUTFORMAT
    NAME "SVG"
    DRIVER CAIRO/SVG
    MIMETYPE "image/svg+xml"
    IMAGEMODE RGB
    EXTENSION "svg"
  END

  # outputformat used by tiled requests
  OUTPUTFORMAT
    NAME "mvt"
    DRIVER MVT
    FORMATOPTION "EDGE_BUFFER=20"
    EXTENSION "pbf"
    FORMATOPTION "EXTENT=4096"
  END  

  WEB
    METADATA
      "ows_title"                      "Example"
      "ows_abstract"                   "Service containing a example"
      "ows_keywordlist"                "example,unknown"
      "ows_schemas_location"           "http://schemas.opengis.net"

      "wfs_extent"                     "7 46 10 48"
      "wfs_namespace_prefix"           "example"
      "wfs_namespace_uri"              "http://example.unknown.org"      
      "wfs_maxfeatures"                "1000"
      "wfs_onlineresource"             "http://localhost"

      "wms_getmap_formatlist"          "image/png,image/jpeg,image/png; mode=8bit,image/vnd.jpeg-png,image/vnd.jpeg-png8,image/svg+xml"
      "wms_enable_request"             "* !GetStyles !DescribeLayer"
      "wms_bbox_extended"              "true"
      "wms_namespace_prefix"           "example"
      "wms_namespace_uri"              "http://example.unknown.org"
      "wms_getfeatureinfo_formatlist"  "text/html,text/xml; subtype=gml/3.2.1,text/xml; subtype=gml/3.1.1,application/json,application/json; subtype=geojson"

      "ows_sld_enabled"                "false"
    END
  END

  LAYER
    NAME "bar_cafe"
    STATUS ON
    TYPE POINT
    ## layer debug settings for mapserver, remove comment in lines below to enable
    DEBUG 4

    METADATA
      "wfs_title"                    "example"
      "wfs_abstract"                 "Layer containing the example data"
      "wfs_srs"                      "EPSG:4326 EPSG:3857 EPSG:4258 EPSG:900913 CRS:84"      
      "wfs_extent"                   "7 46 10 48"
      "wfs_bbox_extended"            "true"
      "wfs_enable_request"           "*"
      "wfs_include_items"            "all"
      "wfs_getfeature_formatlist"    "OGRGML3,OGRGML32,GEOJSON,JSON"

      "gml_include_items"            "all"
      "gml_exclude_items"            "id"
      "gml_featureid"                "id"
      "gml_geometries"               "geom"
      "gml_types"                    "auto"

      "wms_title"                    "example"
      "wms_extent"                   "7 46 10 48"    
      "wms_abstract"                 "Layer containing the example data"
      "wms_srs"                      "EPSG:4326 EPSG:3857 EPSG:4258 EPSG:900913 CRS:84"
      "wms_keywordlist"              "example,unknown"
      "wms_include_items"            "all"
    END

    CLASSGROUP "bar_cafe:style"
    CLASS
      NAME "bar_cafe"
      GROUP "bar_cafe:style"
      STYLE
        COLOR 230 0 0
        SIZE 4
        WIDTH 2
        SYMBOL "circle"
      END
    END

    PROJECTION
      "init=epsg:4326"
    END

    CONNECTIONTYPE OGR
    CONNECTION "bar_cafe.geojson"
    DATA "bar_cafe"
    
    DUMP TRUE

  END # LAYER
END # MAP
```

</details>

<details>
<summary>

Daten `mapserver/data/bar_cafe.geojson`

</summary>

```json
{
  "type": "FeatureCollection",
  "name": "bar_cafe",
  "generator": "overpass-ide",
  "copyright": "The data included in this document is from www.openstreetmap.org. The data is made available under ODbL.",
  "timestamp": "2021-05-31T06:32:42Z",
  "features": [
    {
      "type": "Feature",
      "properties": {
        "@id": "node/1355269649",
        "addr:city": "Rapperswil",
        "addr:housenumber": "4",
        "addr:postcode": "8640",
        "addr:street": "Fischmarktplatz",
        "amenity": "cafe",
        "name": "Cafe Rosenstädter",
        "opening_hours": "Mo-Su 07:00-19:00",
        "website": "http://www.rosenstaedter.ch"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8156698,
          47.2258223
        ]
      },
      "id": "node/1355269649"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/1356639640",
        "amenity": "bar",
        "name": "R1-Bar",
        "opening_hours": "24/7"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8161285,
          47.2262279
        ]
      },
      "id": "node/1356639640"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/1369664249",
        "addr:city": "Rapperswil SG",
        "addr:housenumber": "15",
        "addr:postcode": "8640",
        "addr:street": "Hauptplatz",
        "amenity": "cafe",
        "cuisine": "regional",
        "name": "KaffeeKlatsch",
        "opening_hours": "Mo-Sa 07:00-18:00; Su 08:00-18:00",
        "operator": "Kaffeeklatsch Rapperswil GmbH",
        "phone": "+41 55 210 75 00",
        "website": "www.kaffee-klatsch.ch/",
        "wheelchair": "limited"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8166433,
          47.2266829
        ]
      },
      "id": "node/1369664249"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/1647465057",
        "amenity": "cafe",
        "brand": "Starbucks",
        "brand:wikidata": "Q37158",
        "brand:wikipedia": "en:Starbucks",
        "cuisine": "coffee_shop",
        "name": "Starbucks",
        "opening_hours": "Mo-Fr 07:30-19:30; Sa 08:00-19:30; Su 10:00-19:00",
        "takeaway": "yes",
        "website": "http://www.starbucks.ch/"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8184058,
          47.2269331
        ]
      },
      "id": "node/1647465057"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/1647470027",
        "addr:city": "Rapperswil SG",
        "addr:housenumber": "24",
        "addr:postcode": "8640",
        "addr:street": "Untere Bahnhofstrasse",
        "amenity": "cafe",
        "name": "Wick",
        "opening_hours": "Mo-Fr 06:30-18:30; Sa 08:00-17:00",
        "shop": "bakery",
        "website": "https://baeckerei-wick.ch/",
        "wheelchair": "no"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8187894,
          47.2258729
        ]
      },
      "id": "node/1647470027"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/2345475865",
        "amenity": "bar",
        "name": "ZAK Jona",
        "website": "http://www.zak-jona.ch/"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.834316,
          47.2320348
        ]
      },
      "id": "node/2345475865"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/2393629816",
        "amenity": "cafe",
        "cuisine": "regional",
        "name": "Räber",
        "opening_hours": "Mo-Fr 6:00-18:30; Sa 7:00-17:00; Su 7:30-17:00",
        "website": "http://raebergmacht.ch",
        "wheelchair": "yes"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8323772,
          47.2221617
        ]
      },
      "id": "node/2393629816"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/2394806486",
        "addr:housenumber": "29",
        "addr:postcode": "8645",
        "addr:street": "St. Gallerstrasse",
        "amenity": "cafe",
        "level": "0",
        "name": "Räber",
        "opening_hours": "Mo-Fr 05:00-18:30; Sa 06:00-17:00; Su 06:30-17:00",
        "shop": "bakery",
        "website": "http://www.raebergemacht.ch",
        "wheelchair": "yes"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8365037,
          47.2301987
        ]
      },
      "id": "node/2394806486"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/2851961628",
        "amenity": "cafe",
        "name": "Wick",
        "opening_hours": "Mo-Fr 05:00-18:30; Sa 05:00-16:00; Su 07:00-13:00",
        "opening_hours:covid19": "Mo-Fr 05:00-18:30; Sa 05:00-16:00; Su 07:00-13:00",
        "shop": "bakery",
        "website": "http://www.baeckerei-wick.ch/wege-zum-gl%C3%BCck/rapperswil-hauptgesch%C3%A4ft/"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8286305,
          47.2292486
        ]
      },
      "id": "node/2851961628"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/3421889385",
        "addr:city": "Rapperswil SG",
        "addr:housenumber": "11",
        "addr:postcode": "8640",
        "addr:street": "Marktgasse",
        "amenity": "cafe",
        "name": "Café good",
        "opening_hours": "Tu-Fr 09:00-18:00; Sa 10:00-18:00; Su 10:00-18:00",
        "outdoor_seating": "yes",
        "phone": "+41 77 417 12 74",
        "website": "http://cafegood.ch/",
        "wheelchair": "no"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8155116,
          47.2262657
        ]
      },
      "id": "node/3421889385"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/4523300294",
        "addr:housenumber": "7",
        "addr:postcode": "8645",
        "addr:street": "Meienbergstrasse",
        "amenity": "cafe",
        "name": "Bar & Gartenbeiz Schüür",
        "name:de": "Bar & Gartenbeiz Schüür",
        "opening_hours": "Mo-Fr 17:00-24:00; Sa 12:00-24:00",
        "website": "http://www.schüür-kempraten.ch"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.820461,
          47.2364207
        ]
      },
      "id": "node/4523300294"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/4846486054",
        "addr:housenumber": "29",
        "addr:street": "Engelplatz;Herrenberg",
        "amenity": "cafe",
        "description": "Second Hand Bistro - everything here can be bought!",
        "name": "inä",
        "opening_hours": "We,Th,Sa 09:00-19:00; Fr 09:00-21:00; Su 10:30-18:00",
        "website": "http://www.inae.ch/"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8185559,
          47.2280196
        ]
      },
      "id": "node/4846486054"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/5143669432",
        "addr:city": "Jona",
        "addr:housenumber": "9",
        "addr:postcode": "8645",
        "addr:street": "Bühlstrasse",
        "alt_name": "Steiner Beck Jona",
        "amenity": "cafe",
        "branch": "Jona",
        "name": "Steiner Beck",
        "opening_hours": "Mo-Fr 06:00-18:30; Sa 07:00-16:00; PH off",
        "operator": "Steiner-Beck AG",
        "shop": "bakery",
        "website": "https://www.steiner-beck.ch/filialen/jona/",
        "wheelchair": "yes"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8356069,
          47.230053
        ]
      },
      "id": "node/5143669432"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/5876018982",
        "amenity": "cafe",
        "changing_table": "no",
        "name": "Ricardos",
        "opening_hours": "Tu-Su 08:30-19:00"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8188236,
          47.2283111
        ]
      },
      "id": "node/5876018982"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/6071865330",
        "amenity": "cafe",
        "changing_table": "yes",
        "level": "0",
        "name": "Cafe Balm",
        "opening_hours": "Mo-Tu,Th 09:00-17:00; We,Fr 09:00-19:45; Sa off; Su,PH 13:00-17:00",
        "outdoor_seating": "yes",
        "toilets": "yes",
        "toilets:access": "customers",
        "toilets:wheelchair": "designated",
        "wheelchair": "designated"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8087229,
          47.2459906
        ]
      },
      "id": "node/6071865330"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/6514180343",
        "addr:city": "Rapperswil",
        "addr:housenumber": "4",
        "addr:postcode": "8640",
        "addr:street": "Strehlgasse",
        "amenity": "cafe",
        "name": "Café Hintergasse",
        "opening_hours": "Mo-Sa 07:00-19:00",
        "outdoor_seating": "yes"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.815822,
          47.2267321
        ]
      },
      "id": "node/6514180343"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/7567685234",
        "access": "private",
        "addr:city": "Rapperswil",
        "addr:housenumber": "8",
        "addr:postcode": "8640",
        "addr:street": "Fischmarktplatz",
        "amenity": "cafe",
        "name": "Yachtclub",
        "operator": "Yacht Club Rapperswil",
        "website": "https://www.ycr.ch/"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8160013,
          47.2255371
        ]
      },
      "id": "node/7567685234"
    },
    {
      "type": "Feature",
      "properties": {
        "@id": "node/7598118306",
        "addr:city": "Rapperswil",
        "addr:housenumber": "42",
        "addr:postcode": "8640",
        "addr:street": "Schmiedgasse",
        "amenity": "bar",
        "name": "Gourmet Ibérico",
        "shop": "wine",
        "website": "https://www.gourmet-iberico.ch/"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [
          8.8181302,
          47.2274263
        ]
      },
      "id": "node/7598118306"
    }
  ]
}

```
</details>

<details>
<summary>

`docker-compose.yml`

</summary>

```yml
services:
...
  mapserver:
    image: pdok/mapserver:latest
    volumes:
      - ./mapserver/data:/srv/data
    environment:
      MS_MAPFILE: /srv/data/bar_cafe.map
    ports:
      - "8585:80"
```

</details>

<details>
<summary>

`mapbender_shared/FOSSGIS_Demo.yml` Dienst einbinden

</summary>

```yml
parameters:
    application:
        fossgis_demo:
        ...
        layersets:
            layersets:
                main:
                    # Diesen Dienst hinzufügen
                    bar_cafe:
                        class: Mapbender\WmsBundle\Entity\WmsInstance
                        url: http://localhost:8585/?request=getcapabilities&service=wms
                        version: 1.3.0
                        title: Bars and Cafes
                        layers:
                            1: { name: bar_cafe, title: Bars and Cafes, visible: true }
                        visible: true
                        format: image/png
                        transparent: true
                        opacity: 100
            ...
```

</details>

```shell
# Mapserver starten
docker-compose up -d mapserver
```

[Demo-Request](http://localhost:8585/?request=getcapabilities&service=wms&_signature=21%3AwjgfXKjcj4_SAg_98raFHv9bhrI&SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&FORMAT=image%2Fpng&TRANSPARENT=true&LAYERS=bar_cafe&STYLES=&_OLSALT=0.08847621550740303&CRS=EPSG%3A4326&WIDTH=2161&HEIGHT=495&BBOX=47.209380851277636%2C8.762149736270743%2C47.24050753812575%2C8.898038161076354)

Cache löschen nicht vergessen!

```shell
docker-compose exec --user www-data php php /var/www/mapbender/app/console cache:clear --env=prod 
```

```
📦workshop
 ┣ 📂mapbender_shared
 ┃ ┣ 📜FOSSGIS_Demo.yml
 ┃ ┣ 📜FOSSGIS_Logo_2021.png
 ┃ ┣ 📜mapbender-setup.sh
 ┃ ┗ 📜parameters.yml
 ┣ 📂mapserver
 ┃ ┗ 📂data
 ┃ ┃ ┣ 📜bar_cafe.geojson
 ┃ ┃ ┗ 📜bar_cafe.map
 ┣ 📂nginx
 ┃ ┗ 📜mapbender.conf
 ┣ 📂php
 ┃ ┗ 📜Dockerfile
 ┣ 📂volumes
 ┃ ┣ 📂postgresql
 ┃ ┃ ┗ ... 
 ┃ ┣ mapbender
 ┃ ┃ ┗ ... 
 ┗ 📜docker-compose.yml
```

## 7. Redis Session Handler

PHP-Erweiterung für Redis installieren.

`php/Dockerfile`

```Docker
...
RUN pecl install -o -f redis \
	&& docker-php-ext-enable redis
...
```

```shell
# PHP-Container neu bauen
docker-compose build php
```

`docker-compose.yml`

```yml
services:
...
  redis:
    image: redis:alpine
```

`mapbender/src/App/NativeRedisSessionHandler.php`

```php
<?php

namespace App;

use \Symfony\Component\HttpFoundation\Session\Storage\Handler\NativeSessionHandler;

class NativeRedisSessionHandler extends NativeSessionHandler
 {
    /**
    * Constructor.
    *
    * @param string $savePath Path of redis server.
    */
    public function __construct($savePath = "", $maxlifetime = 432000)
    {
        if (!extension_loaded('redis')) {
            throw new \RuntimeException('PHP does not have "redis" session module registered');
        }

        if ("" === $savePath) {
            $savePath = ini_get('session.save_path');
        }

        if ("" === $savePath) {
            $savePath = "tcp://localhost:6379"; //guess path
        }

        ini_set('session.save_handler', 'redis');
        ini_set('session.save_path', $savePath);
        ini_set('session.gc_maxlifetime', $maxlifetime);
    }
 }
```

`mapbender/app/config/config.yml`

```yml
services:
    ...
    session_handler_redis:
            class: App\NativeRedisSessionHandler
            arguments: ['tcp://redis:6379', 432000]


framework:
    ...
    session:
        handler_id: session_handler_redis
```

Cache löschen nicht vergessen

```shell
docker-compose exec php php /var/www/mapbender/app/console cache:clear --env=prod 
```

Redis CLI aufrufen und testen

```shell
docker-compose exec redis redis-cli

127.0.0.1:6379> select 0
# OK
127.0.0.1:6379> dbsize
# (integer) 999
```

## Finale `docker-compose.yml`

```yml
version: "3.1"
services:
  php:
    build: ./php
    volumes:
      - ./volumes/www/:/var/www
      - ./mapbender_shared/mapbender-setup.sh:/tmp/mapbender-setup:ro
      - ./mapbender_shared/FOSSGIS_Demo.yml:/var/www/mapbender/app/config/applications/FOSSGIS_Demo.yml
    #   - ./mapbender_shared/FOSSGIS_Logo_2021.png:/var/www/mapbender/web/uploads/fossgis_demo/screenshot.png
      - ./mapbender_shared/parameters.yml:/var/www/mapbender/app/config/parameters.yml
    command: bash -c "/tmp/mapbender-setup && docker-php-entrypoint php-fpm"
    environment:
        DATABASE_DRIVER: pdo_pgsql
        DATABASE_HOST: postgis
        DATABASE_PORT: 5432
        DATABASE_PATH: "~"
        DATABASE_NAME: mapbender
        DATABASE_USER: mb_db_user
        DATABASE_PASSWORD: mb_db_pass
#   apache:
#     build: ./apache
#     ports:
#       - "8081:80"
#     volumes:
#       - ./volumes/www/:/var/www
#       - ./apache/mapbender.conf:/etc/apache2/sites-enabled/mapbender.conf
  nginx:
    image: nginx:latest
    ports:
      - "8081:80"
    volumes:
      - ./volumes/www/:/var/www
      - ./nginx/mapbender.conf:/etc/nginx/conf.d/default.conf
  postgis:
    image: postgis/postgis:12-3.1-alpine
    environment:
        POSTGRES_DB: mapbender
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres_pass
    volumes:
        - ./volumes/postgresql/:/var/lib/postgresql/data
  mapserver:
    image: pdok/mapserver:latest
    volumes:
        - ./mapserver/data:/srv/data
    environment:
        MS_MAPFILE: /srv/data/bar_cafe.map
    ports:
        - "8585:80"
  redis:
    image: redis:alpine
    

```

Ordnerstruktur

```
📦workshop
 ┣ 📂apache
 ┃ ┣ 📜Dockerfile
 ┃ ┗ 📜mapbender.conf
 ┣ 📂apache-mod-php
 ┃ ┣ 📜Dockerfile
 ┃ ┗ 📜mapbender.conf
 ┣ 📂mapbender_shared
 ┃ ┣ 📜FOSSGIS_Demo.yml
 ┃ ┣ 📜FOSSGIS_Logo_2021.png
 ┃ ┣ 📜mapbender-setup.sh
 ┃ ┗ 📜parameters.yml
 ┣ 📂mapserver
 ┃ ┗ 📂data
 ┃ ┃ ┣ 📜bar_cafe.geojson
 ┃ ┃ ┗ 📜bar_cafe.map
 ┣ 📂nginx
 ┃ ┗ 📜mapbender.conf
 ┣ 📂php
 ┃ ┗ 📜Dockerfile
 ┣ 📂volumes
 ┃ ┣ 📂postgresql
 ┃ ┃ ┗ ... 
 ┃ ┣ mapbender
 ┃ ┃ ┗ ... 
 ┗ 📜docker-compose.yml
```
