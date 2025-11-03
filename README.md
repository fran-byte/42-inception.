# 42-inception


## Resumen del Proyecto Inception

El proyecto Inception implementa una infraestructura WordPress completa usando Docker con arquitectura de microservicios, donde cada componente (MariaDB, WordPress, Nginx) se ejecuta en contenedores Alpine Linux 3.21 aislados que se comunican a través de una red bridge privada.<cite />

### Arquitectura de Servicios Obligatorios

#### MariaDB - Base de Datos

**Construcción de la Imagen**: El contenedor MariaDB se construye desde Alpine Linux 3.21, instalando los paquetes `mariadb` y `mariadb-client`. [1](#1-0)  Durante la construcción, se crean los directorios necesarios (`/run/mysqld` y `/var/lib/mysql`) con permisos del usuario `mysql`.<cite />

**Inicialización**: El script `entrypoint.sh` gestiona la inicialización completa del servidor de base de datos. Primero lee las credenciales desde Docker secrets montados en `/run/secrets/mariadb_root_password` y `/run/secrets/mariadb_user_password`.<cite /> Si la base de datos no existe, ejecuta `mariadb-install-db --user=mysql --datadir=/var/lib/mysql` para crear la estructura inicial.<cite /> Luego genera un script SQL temporal que configura la contraseña root, crea la base de datos `wordpress`, y establece el usuario `db_user` con privilegios completos mediante `CREATE USER` y `GRANT ALL PRIVILEGES`.<cite /> Finalmente, inicia el servidor con `mysqld --user=mysql` escuchando en el puerto 3306.<cite />

**Persistencia**: Los datos se almacenan en el volumen Docker montado en `/var/lib/mysql`, garantizando que la información persista entre reinicios de contenedores. [2](#1-1) 

#### WordPress - Aplicación PHP

**Construcción de la Imagen**: El Dockerfile de WordPress es el más complejo del proyecto. Comienza instalando PHP 8.3 con el módulo FPM y 12 extensiones esenciales: `php83-mysqli` (conexión MySQL), `php83-curl` (peticiones HTTP), `php83-dom` (manipulación XML/HTML), `php83-json` (procesamiento JSON), `php83-mbstring` (strings multibyte), `php83-xml` (procesamiento XML), `php83-zip` (archivos comprimidos), `php83-phar` (archivos PHP), `php83-session` (sesiones), `php83-openssl` (criptografía), y `php83-tokenizer` (análisis de código). [3](#1-2) 

Además instala herramientas del sistema: `wget` y `curl` para descargas, y `mariadb-client` para conectividad con la base de datos. [4](#1-3)  Crea los directorios `/var/www/html` (raíz web) y `/run/php` (sockets PHP-FPM) con propietario `nobody:nobody` y permisos 755. [5](#1-4) 

Durante la construcción, descarga WordPress desde `https://wordpress.org/latest.tar.gz`, lo extrae a `/tmp/wordpress/`, copia los archivos a `/var/www/html/`, y establece permisos recursivos (755 para directorios, 644 para archivos). [6](#1-5) 

**Configuración PHP-FPM**: Copia el archivo `conf/www.conf` a `/etc/php83/php-fpm.d/www.conf` que configura el pool de procesos PHP-FPM. [7](#1-6)  También copia tres scripts de inicialización: `install.sh` (instalación de WP-CLI), `entrypoint.sh` (inicialización del contenedor), e `init-users.php` (creación de usuarios WordPress). [8](#1-7) 

**Proceso de Inicialización**: El `entrypoint.sh` ejecuta una secuencia de 7 pasos críticos:

1. **Gestión de Permisos**: Establece `nobody:nobody` como propietario recursivo de `/var/www/html` con permisos 755 para directorios y 644 para archivos mediante `find` y `chmod`. [9](#1-8) 

2. **Descarga Condicional**: Si el volumen `/var/www/html` está vacío (primera ejecución), descarga `latest.tar.gz` desde wordpress.org usando `wget -q --no-check-certificate`, extrae el contenido a `/tmp/`, copia los archivos con `cp -a` (preservando atributos), y limpia archivos temporales. [10](#1-9) 

3. **Espera de MariaDB**: Lee la contraseña de base de datos desde `/run/secrets/mariadb_user_password` y ejecuta un bucle de reintentos (máximo 15 intentos con intervalos de 2 segundos) usando `mysql -h mariadb -u db_user` para verificar conectividad con `SELECT 1;`. [11](#1-10)  Si falla después de 15 intentos, el contenedor termina con código de error 1. [12](#1-11) 

4. **Configuración WordPress**: Verifica si WordPress está instalado con `wp core is-installed --allow-root`. Si no lo está, ejecuta `wp config create` para generar `wp-config.php` con los parámetros de conexión: `--dbhost=mariadb`, `--dbname=wordpress`, `--dbuser=db_user`, y la contraseña leída del secret. [13](#1-12) 

5. **Instalación del Core**: Ejecuta `wp core install` con parámetros del entorno: URL del sitio (`https://${DOMAIN_NAME}`), título, usuario administrador (`${WORDPRESS_ADMIN_USER}`), contraseña desde `/run/secrets/wp_manager_password`, email del administrador, y `--skip-email` para evitar envío de correos. [14](#1-13) 

6. **Configuración de Permalinks**: Establece la estructura de URLs amigables con `wp rewrite structure '/%postname%/'` usando el flag `--hard` para forzar la escritura del archivo `.htaccess`, seguido de `wp rewrite flush --hard` para limpiar las reglas de reescritura. [15](#1-14) 

7. **Creación de Usuarios Adicionales**: Ejecuta el script PHP `init-users.php` si existe, usando `|| true` para evitar que errores detengan el proceso. [16](#1-15) 

Finalmente, vuelve a configurar permalinks (por si hubo cambios) y arranca PHP-FPM en modo foreground con `exec php-fpm83 -F` en el puerto 9000. [17](#1-16) 

**Gestión de Usuarios WordPress**: El sistema implementa un mecanismo de dos etapas para crear usuarios. Durante `wp core install`, WP-CLI crea automáticamente el usuario administrador usando las variables de entorno `WORDPRESS_ADMIN_USER`, `WORDPRESS_ADMIN_EMAIL` y la contraseña desde `/run/secrets/wp_manager_password`. [14](#1-13) 

El script `init-users.php` maneja la creación del usuario editor. Define la función `create_user_if_not_exists()` que verifica la existencia del usuario con `username_exists()` y del email con `email_exists()` antes de crear el usuario con `wp_create_user()`. Si la creación es exitosa, obtiene el objeto `WP_User` y asigna el rol con `$user->set_role()`. [18](#1-17)  Lee las credenciales del editor desde variables de entorno (`WORDPRESS_REGULAR_USER`, `WORDPRESS_REGULAR_EMAIL`) y el secret `/run/secrets/wp_editor_password`, luego invoca la función para crear ambos usuarios (administrador y editor) con sus respectivos roles. [19](#1-18) 

#### Nginx - Servidor Web y Reverse Proxy

**Construcción de la Imagen**: El Dockerfile de Nginx es el más simple. Actualiza el índice de paquetes con `apk update`, instala `nginx` y `openssl`, copia la configuración principal `nginx.conf` a `/etc/nginx/nginx.conf` y la configuración del servidor `default.conf` a `/etc/nginx/conf.d/default.conf`.<cite /> Expone el puerto 443 para HTTPS y ejecuta `nginx -g 'daemon off;'` para mantener el proceso en foreground.<cite />

**Configuración HTTPS**: El servidor escucha en el puerto 443 con SSL habilitado para el dominio `frromero.42.fr`. Los certificados se montan desde el host: `/etc/ssl/certs/frromero.42.fr.crt` (certificado público) y `/etc/ssl/private/frromero.42.fr.key` (clave privada). [20](#1-19)  Soporta TLS 1.2 y 1.3 con timeout de sesión de 10 minutos. [21](#1-20) 

**Resolución DNS**: Configura el resolver DNS de Docker en `127.0.0.11` con validez de 30 segundos para la resolución de nombres de servicios dentro de la red `inception_net`. [22](#1-21) 

**Servicio de Archivos Estáticos**: Define `/var/www/html` como raíz del documento con `index.php` e `index.html` como archivos índice. [23](#1-22)  La directiva `location /` implementa el patrón de WordPress con `try_files $uri $uri/ /index.php?$args`, intentando servir el archivo directamente, luego como directorio, y finalmente redirigiendo a `index.php`. [24](#1-23)  Desactiva el caché del navegador con headers `Cache-Control: 'no-store, no-cache'`, `if_modified_since off`, `expires off`, y `etag off`. [25](#1-24) 

**Procesamiento PHP via FastCGI**: La directiva `location ~ \.php$` captura todas las peticiones a archivos PHP. Usa `fastcgi_split_path_info` para separar el script del path info, reenvía la petición a `wordpress:9000` con `fastcgi_pass`, incluye parámetros FastCGI estándar, y establece `SCRIPT_FILENAME` y `PATH_INFO` para que PHP-FPM procese correctamente la petición. [26](#1-25) 

**Proxy de Servicios Bonus**: Configura dos ubicaciones adicionales para servicios bonus. `/adminer` hace proxy a `adminer:8090` y `/web/` a `web:8080`, ambos con headers de proxy estándar (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) y manejo de errores que convierte códigos 502/503/504 en 404. [27](#1-26) 

### Comunicación Entre Servicios

**Flujo de Datos**: Las peticiones HTTPS del cliente llegan a Nginx en el puerto 443. Nginx determina el tipo de contenido: archivos estáticos (CSS, JS, imágenes) se sirven directamente desde `/var/www/html` (volumen compartido con WordPress), mientras que archivos PHP se reenvían a `wordpress:9000` mediante el protocolo FastCGI.<cite /> WordPress procesa la petición PHP, ejecuta consultas a la base de datos conectándose a `mariadb:3306` usando la extensión `mysqli`, y devuelve la respuesta a Nginx, que la transmite al cliente.<cite />

**Resolución de Nombres**: La red bridge `inception_net` proporciona DNS automático donde cada contenedor puede resolver nombres de servicio (`mariadb`, `wordpress`, `nginx`) a sus direcciones IP internas mediante el servidor DNS embebido de Docker en `127.0.0.11`. [28](#1-27) 

**Volumen

Wiki pages you might want to explore:
- [Core Services (fran-byte/backup-inception_v3)](/wiki/fran-byte/backup-inception_v3#4)
- [Building and Rebuilding Images (fran-byte/backup-inception_v3)](/wiki/fran-byte/backup-inception_v3#7.2)

### Citations

**File:** srcs/docker-compose.yml (L14-14)
```yaml
      - ${VOLUMES_ROOT}/db:/var/lib/mysql
```

**File:** srcs/requirements/wordpress/Dockerfile (L3-16)
```text
RUN apk add --no-cache \
        php83 \
        php83-fpm \
        php83-mysqli \
        php83-curl \
        php83-dom \
        php83-json \
        php83-mbstring \
        php83-xml \
        php83-zip \
        php83-phar \
        php83-session \
        php83-openssl \
        php83-tokenizer \
```

**File:** srcs/requirements/wordpress/Dockerfile (L17-19)
```text
        wget \
        curl \
        mariadb-client \
```

**File:** srcs/requirements/wordpress/Dockerfile (L20-22)
```text
    && mkdir -p /var/www/html /run/php \
    && chown -R nobody:nobody /var/www/html /run/php \
    && chmod -R 755 /var/www/html
```

**File:** srcs/requirements/wordpress/Dockerfile (L24-30)
```text
RUN wget https://wordpress.org/latest.tar.gz -O /tmp/wordpress.tar.gz \
    && tar -xzf /tmp/wordpress.tar.gz -C /tmp/ \
    && cp -a /tmp/wordpress/. /var/www/html/ \
    && rm -rf /tmp/wordpress /tmp/wordpress.tar.gz \
    && chown -R nobody:nobody /var/www/html \
    && find /var/www/html -type d -exec chmod 755 {} \; \
    && find /var/www/html -type f -exec chmod 644 {} \;
```

**File:** srcs/requirements/wordpress/Dockerfile (L32-32)
```text
COPY conf/www.conf /etc/php83/php-fpm.d/www.conf
```

**File:** srcs/requirements/wordpress/Dockerfile (L33-35)
```text
COPY tools/install.sh /usr/local/bin/install.sh
COPY tools/entrypoint.sh /usr/local/bin/entrypoint.sh
COPY tools/init-users.php /usr/local/bin/init-users.php
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L4-7)
```shellscript
# Permissions
chown -R nobody:nobody /var/www/html
chmod -R 755 /var/www/html
find /var/www/html -type f -exec chmod 644 {} \;
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L9-16)
```shellscript
# Copy WordPress if volume is empty
if [ -z "$(ls -A /var/www/html)" ]; then
    wget -q --no-check-certificate https://wordpress.org/latest.tar.gz -O /tmp/wordpress.tar.gz
    tar -xzf /tmp/wordpress.tar.gz -C /tmp/
    cp -a /tmp/wordpress/. /var/www/html/
    rm -rf /tmp/wordpress /tmp/wordpress.tar.gz
    chown -R nobody:nobody /var/www/html
fi
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L18-27)
```shellscript
# Wait for MariaDB
DB_PASSWORD=$(cat /run/secrets/mariadb_user_password)

for i in $(seq 1 15); do
    if mysql -h mariadb -u db_user -p"$DB_PASSWORD" -e "SELECT 1;" &> /dev/null; then
        break
    fi
    sleep 2
    [ $i -eq 15 ] && echo "MariaDB connection failed" && exit 1
done
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L30-37)
```shellscript
if ! wp core is-installed --allow-root --path=/var/www/html 2>/dev/null; then
    [ ! -f /var/www/html/wp-config.php ] && wp config create \
        --dbhost=mariadb \
        --dbname=wordpress \
        --dbuser=db_user \
        --dbpass="$DB_PASSWORD" \
        --allow-root \
        --path=/var/www/html
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L39-47)
```shellscript
    wp core install \
        --url="https://${DOMAIN_NAME}" \
        --title="frromero WordPress" \
        --admin_user="${WORDPRESS_ADMIN_USER}" \
        --admin_password="$(cat /run/secrets/wp_manager_password)" \
        --admin_email="${WORDPRESS_ADMIN_EMAIL}" \
        --skip-email \
        --allow-root \
        --path=/var/www/html
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L49-51)
```shellscript
    # CONF PERMALINKS
    wp rewrite structure '/%postname%/' --hard --allow-root --path=/var/www/html
    wp rewrite flush --hard --allow-root --path=/var/www/html
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L53-53)
```shellscript
    [ -f /usr/local/bin/init-users.php ] && php /usr/local/bin/init-users.php || true
```

**File:** srcs/requirements/wordpress/tools/entrypoint.sh (L56-59)
```shellscript
wp rewrite structure '/%postname%/' --hard --allow-root --path=/var/www/html
wp rewrite flush --hard --allow-root --path=/var/www/html

```

**File:** srcs/requirements/wordpress/tools/init-users.php (L6-15)
```php
// Function: create_user_if_not_exists, Creates a user only if it does not exist
function create_user_if_not_exists($username, $email, $password, $role) {
    if (!username_exists($username) && !email_exists($email)) {
        $user_id = wp_create_user($username, $password, $email);
        if (!is_wp_error($user_id)) {
            $user = new WP_User($user_id);
            $user->set_role($role);
            error_log("User '$username' created with role '$role'");
        }
    }
```

**File:** srcs/requirements/wordpress/tools/init-users.php (L18-30)
```php
// Read environment variables and secrets
$admin_user = getenv('WORDPRESS_ADMIN_USER') ?: 'wp_manager_user';
$admin_pass = trim(file_get_contents('/run/secrets/wp_manager_password'));
$admin_email = getenv('WORDPRESS_ADMIN_EMAIL') ?: 'manager@42.fr';

$editor_user = getenv('WORDPRESS_REGULAR_USER') ?: 'wp_editor_user';
$editor_pass = trim(file_get_contents('/run/secrets/wp_editor_password'));
$editor_email = getenv('WORDPRESS_REGULAR_EMAIL') ?: 'editor@42.fr';


// Create users if they do not exist
create_user_if_not_exists($admin_user, $admin_email, $admin_pass, 'administrator');
create_user_if_not_exists($editor_user, $editor_email, $editor_pass, 'editor');
```

**File:** srcs/requirements/nginx/conf/default.conf (L2-6)
```text
    listen 443 ssl;
    server_name frromero.42.fr;

    ssl_certificate     /etc/ssl/certs/frromero.42.fr.crt;
    ssl_certificate_key /etc/ssl/private/frromero.42.fr.key;
```

**File:** srcs/requirements/nginx/conf/default.conf (L7-8)
```text
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_session_timeout 10m;
```

**File:** srcs/requirements/nginx/conf/default.conf (L10-11)
```text
    # Resolver DNS de Docker
    resolver 127.0.0.11 valid=30s;
```

**File:** srcs/requirements/nginx/conf/default.conf (L13-14)
```text
    root /var/www/html;
    index index.php index.html;
```

**File:** srcs/requirements/nginx/conf/default.conf (L17-18)
```text
    location / {
        try_files $uri $uri/ /index.php?$args;
```

**File:** srcs/requirements/nginx/conf/default.conf (L19-23)
```text
        add_header Last-Modified $date_gmt;
        add_header Cache-Control 'no-store, no-cache';
        if_modified_since off;
        expires off;
        etag off;
```

**File:** srcs/requirements/nginx/conf/default.conf (L27-34)
```text
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
```

**File:** srcs/requirements/nginx/conf/default.conf (L36-62)
```text
    # Adminer
    location /adminer {
        set $adminer_upstream adminer:8090;
        proxy_pass http://$adminer_upstream;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_intercept_errors on;
        error_page 502 503 504 =404 /404.html;
    }
            
    # WEB
    location /web/ {
        set $web_upstream web:8080;
        proxy_pass http://$web_upstream/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_set_header X-Forwarded-Ssl on;
        proxy_set_header X-Url-Scheme https;
        proxy_intercept_errors on;
        error_page 502 503 504 =404 /404.html;
    }
```
