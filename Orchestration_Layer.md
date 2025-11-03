# Orchestration Layer

**Docker Compose**  
`(srcs/docker-compose.yml)`  
sirve como motor de orquestación, definiendo:

| Component   | Purpose               | Key Configuration                                           |
|-------------|------------------------|-------------------------------------------------------------|
| Services    | Container definitions  | `build`, `image`, `container_name`, `restart`              |
| Dependencies| Service startup order  | `depends_on: [mariadb]`, `depends_on: [wordpress]`         |
| Networks    | Service connectivity   | `inception_net` bridge network                             |
| Volumes     | Data persistence       | `db`, `wp` named volumes with bind mounts                  |
| Secrets     | Credential injection   | File-based secrets mounted at `/run/secrets/`              |
| Profiles    | Optional services      | `profiles: [bonus]` for Adminer and Web                    |



---

# Dependency Order

La cadena de dependencias está definida en  
`srcs/docker-compose.yml`

| Service   | Depends On | Startup Position | Rationale                                           |
|-----------|------------|------------------|-----------------------------------------------------|
| mariadb   | None       | First            | Database must be ready before WordPress            |
| wordpress | mariadb    | Second           | Requires database connection for initialization    |
| nginx     | wordpress  | Third            | Requires WordPress PHP-FPM to be listening         |
| adminer   | None (implicit) | Parallel     | Connects to database at runtime                    |
| web       | None       | Parallel         | Independent static content server                  |

---

# Bridge Network Configuration

El sistema define una única red personalizada tipo *bridge* en la configuración de Docker Compose:

```yaml
networks:
  inception_net:
    driver: bridge
```

### Network Properties

| Property       | Value          | Description                                           |
|----------------|----------------|-------------------------------------------------------|
| Network Name   | `inception_net`| Logical name used in service definitions             |
| Driver Type    | `bridge`       | Docker bridge driver for single-host networking      |
| Scope          | `Local`        | Network exists only on the Docker host               |
| Internal       | `No`           | Allows external connectivity through port mappings   |



---

# Volume Strategy

| Volume Name | Host Path                    | Purpose                  | Sharing                          |
|-------------|------------------------------|--------------------------|----------------------------------|
| `db`        | `${PROJECT_ROOT}/data/db`    | MariaDB data files       | Exclusive to MariaDB            |
| `wp`        | `${PROJECT_ROOT}/data/wp`    | WordPress installation   | Shared: WordPress + Nginx       |

---


# Service Stack Architecture

Los tres servicios principales implementan una arquitectura clásica de aplicación web en tres capas, cada uno ejecutándose en su propio contenedor basado en Alpine Linux:

| Service   | Container Name | Base Image   | Primary Role                                         | Exposed Port         |
|-----------|----------------|--------------|------------------------------------------------------|----------------------|
| Nginx     | `nginx`        | `alpine:3.21`| SSL termination, reverse proxy, static file serving  | 443 (HTTPS)          |
| WordPress | `wordpress`    | `alpine:3.21`| PHP-FPM application server, WordPress core           | 9000 (FastCGI)       |
| MariaDB   | `mariadb`      | `alpine:3.21`| MySQL-compatible database server                     | 3306 (MySQL protocol)|


---

# Dependency Chain

Las relaciones `depends_on` crean el siguiente orden de inicio:

### **MariaDB (`mariadb`)**: *Starts first with no dependencies*
- Inicializa el almacenamiento de la base de datos en `/var/lib/mysql`
- Crea la base de datos de WordPress y las cuentas de usuario
- Comienza a aceptar conexiones en el puerto **3306**

---

### **WordPress (`wordpress`)**: *Starts after MariaDB*
- Espera a que MariaDB esté aceptando conexiones
- Descarga y configura los archivos core de WordPress
- Realiza la instalación del esquema de base de datos
- Crea los usuarios administrador y editor de WordPress
- Inicia PHP-FPM escuchando en el puerto **9000**

---

### **Nginx (`nginx`)**: *Starts last after WordPress*
- Carga los certificados SSL desde `/etc/ssl/certs/` y `/etc/ssl/private/`
- Configura el proxy FastCGI hacia `wordpress:9000`
- Comienza a servir tráfico HTTPS en el puerto **443**

---



# Nginx Service – Key Responsibilities

| Responsibility         | Description                                                                                           |
|------------------------|-------------------------------------------------------------------------------------------------------|
| SSL/TLS Termination    | Accepts encrypted HTTPS connections, decrypts traffic, and forwards requests to backend services over unencrypted internal connections |
| Reverse Proxy          | Routes requests to appropriate backend services based on URL path patterns                           |
| Static File Server     | Directly serves static WordPress assets (CSS, JavaScript, images) without proxying to PHP-FPM        |
| Load Distribution      | Acts as the single gateway for multiple backend services (WordPress, Adminer, Web)                   |
| DNS Resolution         | Resolves service names to container IPs within the Docker network                                     |





## Server Block Structure

El archivo `default.conf` define un único bloque de servidor que escucha en el puerto **443** con SSL habilitado:

| Configuration Section   | Purpose                                           |  
|-------------------------|---------------------------------------------------|
| `listen` directive      | Binds to port 443 with SSL protocol               |     
| `server_name`           | Defines the domain name `frromero.42.fr`          |     
| `ssl_certificate` paths | Points to SSL certificate and private key files   |  
| `ssl_protocols`         | Restricts to TLSv1.2 and TLSv1.3                  |     
| `resolver`              | Configures Docker's internal DNS (`127.0.0.11`)   |    
| `root`                  | Sets document root to `/var/www/html`             |     
| `location` blocks       | Defines routing rules for different URL paths     |  


---

## PHP Processing Location (`~ \.php$`)

Este bloque `location` usa una expresión regular para hacer coincidir todas las solicitudes de archivos PHP:

```nginx
location ~ \.php$ {
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass wordpress:9000;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
}
```

### Configuration Elements

| Directive                     | Purpose                                                                 |
|-------------------------------|-------------------------------------------------------------------------|
| `fastcgi_split_path_info`     | Parsea la URI para extraer el nombre del script PHP y la información de ruta |
| `fastcgi_pass wordpress:9000` | Redirige la solicitud al PHP-FPM del contenedor WordPress en el puerto 9000 |
| `fastcgi_index`               | Archivo por defecto cuando se solicita un directorio                   |
| `fastcgi_param SCRIPT_FILENAME` | Establece la ruta absoluta al script PHP                             |
| `fastcgi_param PATH_INFO`     | Pasa la información de ruta a la aplicación PHP                        |

El destino `wordpress:9000` utiliza la resolución de servicios de Docker para obtener la IP del contenedor de WordPress. PHP-FPM escucha en el puerto 9000 dentro del contenedor, recibiendo solicitudes mediante el protocolo FastCGI.

---


## Adminer Proxy Location (`/adminer`)

El bloque `location` para Adminer implementa un proxy inverso HTTP hacia el servicio de gestión de bases de datos incluido en el perfil *bonus*:

```nginx
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
```

### Key Implementation Details

- **Variable Assignment (línea 38)**:  
  `set $adminer_upstream adminer:8090;` asigna el destino upstream a una variable. Este patrón evita que Nginx falle al iniciar si el servicio Adminer no está disponible (cuando el perfil *bonus* no está activado).

- **Proxy Headers (líneas 40–43)**:  
  Configuran cabeceras estándar de proxy inverso para preservar la información original del cliente:
  - `Host`: mantiene la cabecera original del host
  - `X-Real-IP`: pasa la IP real del cliente
  - `X-Forwarded-For`: cadena de IPs de proxy
  - `X-Forwarded-Proto`: protocolo original (https)

- **Error Handling (líneas 45–46)**:  
  Intercepta errores del upstream (`502`, `503`, `504`) y los convierte en respuestas `404`, proporcionando una degradación elegante cuando Adminer no está disponible.



---

## Static Web Proxy Location (`/web/`)

El bloque `location` para el servidor web estático implementa un proxy inverso hacia el servicio HTML incluido en el perfil *bonus*:

```nginx
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

### Notable Differences from Adminer Configuration

- **Trailing Slash en `proxy_pass` (línea 52)**:  
  La URI `http://$web_upstream/` incluye una barra final, lo que indica a Nginx que elimine el prefijo `/web/` al reenviar la solicitud.  
  Por ejemplo, una solicitud a `/web/about.html` se reenvía como `/about.html` al upstream.

- **Cabeceras SSL adicionales (líneas 58–59)**:  
  Se establecen las cabeceras `X-Forwarded-Ssl` y `X-Url-Scheme` para indicar explícitamente que la solicitud original usó HTTPS.

- **Manejo de errores similar**:  
  Coincide con el patrón de degradación elegante de Adminer para servicios *bonus* no disponibles.

---


## Configuration Summary Table

| Location Pattern | Match Type        | Backend Target | Protocol     | Port | Purpose                    |
|------------------|-------------------|----------------|--------------|------|----------------------------|
| `~ \.php$`       | Regular Expression| `wordpress`    | FastCGI      | 9000 | PHP script processing      |
| `/adminer`       | Prefix            | `adminer`      | HTTP Proxy   | 8090 | Database management UI     |
| `/web/`          | Prefix            | `web`          | HTTP Proxy   | 8080 | Static HTML content        |
| `/`              | Prefix            | N/A (`try_files`) | Static/FastCGI | N/A  | WordPress application root |

---


## Certificate Configuration

### Certificate Files

El sistema utiliza un certificado X.509 autofirmado para el dominio `frromero.42.fr`. El certificado y la clave privada se almacenan en el sistema anfitrión y se montan en el contenedor de Nginx.

| File         | Host Path                                 | Container Path                                  | Purpose                                      |
|--------------|--------------------------------------------|--------------------------------------------------|----------------------------------------------|
| Certificate  | `secrets/certs/frromero.42.fr.crt`         | `/etc/ssl/certs/frromero.42.fr.crt`             | Certificado público presentado a los clientes |
| Private Key  | `secrets/certs/frromero.42.fr.key`         | `/etc/ssl/private/frromero.42.fr.key`           | Clave privada para descifrado del certificado |




---

## WP Runtime Dependencies

### Service Dependencies

El servicio de WordPress tiene dependencias estrictas definidas en la configuración de Docker Compose:

| Dependency              | Type              | Purpose                                                  |
|-------------------------|-------------------|----------------------------------------------------------|
| `mariadb`               | Service Dependency| Backend de base de datos para almacenamiento de WordPress|
| `mariadb_user_password`| Docker Secret     | Credenciales para la conexión a la base de datos         |
| `wp_manager_password`   | Docker Secret     | Contraseña del usuario administrador                     |
| `wp_editor_password`    | Docker Secret     | Contraseña del usuario editor                            |
| `wp` Volume             | Named Volume      | Almacenamiento persistente para archivos de WordPress    |
| `inception_net`         | Docker Network    | Comunicación con MariaDB y Nginx                         |

---


## Resumen del WordPress Container Build

### Tabla de resumen

| **Aspecto**                     | **Detalles**                                                                 |
|----------------------------------|------------------------------------------------------------------------------|
| **Imagen Base**                 | `alpine:3.21` 1                                                      |
| **PHP y Extensiones**          | PHP 8.3 con FPM, mysqli, curl, dom, json, mbstring, xml, zip, phar, session, openssl, tokenizer 2 |
| **Utilidades**                 | `wget`, `curl`, `mariadb-client` 3                                   |
| **Directorios Creados**        | `/var/www/html` (WordPress), `/run/php` (PHP-FPM socket) 4           |
| **Descarga WordPress**         | Descarga `latest.tar.gz` desde wordpress.org y extrae a `/var/www/html` 5 |
| **Permisos**                   | Usuario: `nobody:nobody`, Directorios: 755, Archivos: 644 6          |
| **Configuración PHP-FPM**      | Copia `www.conf` a `/etc/php83/php-fpm.d/` 7                          |
| **Scripts de Inicialización**  | `install.sh`, `entrypoint.sh`, `init-users.php` copiados a `/usr/local/bin/` 8 |
| **Puerto Expuesto**            | 9000 (FastCGI) 9                                                      |
| **Entrypoint**                 | `/usr/local/bin/entrypoint.sh` 10                                    |
| **Workdir**                    | `/var/www/html` 11                                                  |

---

### Comandos de Build

| **Comando**             | **Descripción**                                 |
|-------------------------|--------------------------------------------------|
| `make build`            | Build estándar con caché 12            |
| `make rebuild`          | Build sin caché + deploy 13            |
| `make mandatory-up`     | Build + deploy servicios obligatorios 14|

---

### Proceso de Inicialización (`entrypoint.sh`)

El script `entrypoint.sh` ejecuta la siguiente secuencia: 15

1. Establece permisos en `/var/www/html`  
2. Descarga WordPress si el volumen está vacío  
3. Espera conexión a MariaDB (máximo 15 intentos)  
4. Crea `wp-config.php` con credenciales de base de datos  
5. Instala WordPress core con WP-CLI  
6. Configura permalinks  
7. Ejecuta `init-users.php` para crear usuarios adicionales  
8. Inicia `php-fpm83 -F` en foreground  

---

### Notas

El contenedor WordPress se integra con:

- **Nginx**: recibe peticiones FastCGI en el puerto 9000  
- **MariaDB**: conexión a `mariadb:3306`  
- **Docker Secrets**: montados en `/run/secrets/` para credenciales seguras 17
