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

