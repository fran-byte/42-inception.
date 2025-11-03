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

## Bridge Network Configuration

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


