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

