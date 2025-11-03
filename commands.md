
## Tabla de Comandos Docker Compose Directos


### 🚀 Arrancar servicios

| **Acción**               | **Comando Docker Compose**                                      |
|--------------------------|------------------------------------------------------------------|
| Servicios obligatorios   | `docker compose up --build -d mariadb wordpress nginx`          |
| Con servicios bonus      | `docker compose --profile bonus up --build -d`                  |

---

### 🛑 Parar servicios

| **Acción**               | **Comando Docker Compose**              |
|--------------------------|------------------------------------------|
| Todos los servicios      | `docker compose stop`                   |
| Servicio específico      | `docker compose stop mariadb`           |

---

### 📋 Logs

| **Acción**               | **Comando Docker Compose**                      |
|--------------------------|--------------------------------------------------|
| Todos en tiempo real     | `docker compose logs --follow`                 |
| Servicio específico      | `docker compose logs mariadb`                  |
| Últimas N líneas         | `docker compose logs --tail=50 wordpress`      |

---

### 🔁 Build

| **Acción**               | **Comando Docker Compose**                      |
|--------------------------|--------------------------------------------------|
| Con caché                | `docker compose build`                         |
| Sin caché                | `docker compose build --no-cache`              |
| Servicio específico      | `docker compose build --no-cache mariadb`      |

---

### 🧹 Limpieza

| **Acción**               | **Comando Docker Compose**                                      |
|--------------------------|------------------------------------------------------------------|
| Parar y eliminar         | `docker compose down --remove-orphans`                          |
| Con bonus                | `docker compose --profile bonus down --remove-orphans`          |

---



### 🛠️ Acciones principales

| **Acción**         | **Comando**                          |
|--------------------|--------------------------------------|
| **Arrancar servicios** | `make mandatory-up`              |
| **Parar servicios**    | `make stop`                      |
| **Reiniciar**          | `make restart`                   |

---

### 📋 Logs y estado

| **Acción**             | **Comando**               |
|------------------------|---------------------------|
| **Ver logs**           | `make logs`               |
| **Ver contenedores**   | `docker ps`               |

---

### 🔁 Reconstrucción

| **Acción**             | **Comando**               |
|------------------------|---------------------------|
| **Rebuild sin caché**  | `make rebuild`            |
| **Build con caché**    | `make build`              |

---

### 🧭 Acceso a contenedores

| **Acción**             | **Comando**                                 |
|------------------------|---------------------------------------------|
| **Shell MariaDB**      | `docker exec -it mariadb sh`                |
| **Shell WordPress**    | `docker exec -it wordpress sh`              |

---

### 🗄️ Base de datos

| **Acción**                     | **Comando**                                      |
|--------------------------------|--------------------------------------------------|
| **Conectar MySQL**             | `docker exec -it mariadb mysql -u root -p`       |
| **Ver bases de datos**         | `SHOW DATABASES;`                                |
| **Ver tablas**                 | `USE wordpress; SHOW TABLES;`                    |
| **Ver usuarios de WordPress** | `SELECT user_login, user_email FROM wp_users;`   |

---


## ⚡ Secuencia Rápida de Evaluación

```bash
# 1. Limpiar y arrancar
make purge-all && make mandatory-up

# 2. Verificar
docker ps
make logs

# 3. Acceder a BD
docker exec -it mariadb mysql -u root -p
SHOW DATABASES;
USE wordpress;
SELECT user_login FROM wp_users;
exit

# 4. Parar/Reiniciar
make stop
make restart

# 5. Rebuild
make rebuild
```

---

