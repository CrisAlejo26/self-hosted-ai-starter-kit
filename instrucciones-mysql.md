# Instrucciones para Integrar Embeddings en n8n Usando MySQL

Este documento describe los pasos para configurar un entorno Docker con n8n y MySQL, almacenar embeddings en una tabla y definir funciones SQL para realizar búsquedas por similitud. La solución almacena los vectores de embeddings como un arreglo JSON y utiliza una función almacenada para calcular la similitud coseno.

---

## Requisitos Previos

- **Docker** y **Docker Compose** instalados.
- Un editor de texto para modificar archivos (por ejemplo, VSCode).
- Un cliente SQL (por ejemplo, **TablePlus** o **MySQL Workbench**) para ejecutar las consultas en MySQL.
- Un archivo `.env` con las variables de entorno (ver más abajo).

---

## 1. Configuración del Entorno Docker

### Archivo `.env`

Crea un archivo `.env` (o ajusta las variables de entorno según tu configuración) con, por ejemplo:

```env
MYSQL_ROOT_PASSWORD=super-secret-root
MYSQL_DATABASE=n8n
MYSQL_USER=n8n_user
MYSQL_PASSWORD=super-secret-pass
N8N_ENCRYPTION_KEY=super-secret-key
N8N_USER_MANAGEMENT_JWT_SECRET=even-more-secret
```

## Archivo docker-compose.yml

El siguiente ejemplo de Docker Compose utiliza MySQL en lugar de PostgreSQL y configura n8n para conectarse a él. Se asume que se usará la base de datos MySQL para almacenar la información de n8n y, adicionalmente, para almacenar documentos con embeddings.

```yaml
volumes:
  n8n_storage:
  mysql_storage:
  ollama_storage:
  qdrant_storage:

networks:
  demo:

x-n8n: &service-n8n
  image: n8nio/n8n:latest
  networks: ['demo']
  environment:
    - DB_TYPE=mysqldb
    - DB_MYSQLDB_HOST=mysql
    - DB_MYSQLDB_PORT=3306
    - DB_MYSQLDB_DATABASE=${MYSQL_DATABASE}
    - DB_MYSQLDB_USER=${MYSQL_USER}
    - DB_MYSQLDB_PASSWORD=${MYSQL_PASSWORD}
    - N8N_DIAGNOSTICS_ENABLED=false
    - N8N_PERSONALIZATION_ENABLED=false
    - N8N_ENCRYPTION_KEY
    - N8N_USER_MANAGEMENT_JWT_SECRET
    - OLLAMA_HOST=ollama:11434

x-ollama: &service-ollama
  image: ollama/ollama:latest
  container_name: ollama
  networks: ['demo']
  restart: unless-stopped
  ports:
    - 11434:11434
  volumes:
    - ollama_storage:/root/.ollama

x-init-ollama: &init-ollama
  image: ollama/ollama:latest
  networks: ['demo']
  container_name: ollama-pull-llama
  volumes:
    - ollama_storage:/root/.ollama
  entrypoint: /bin/sh
  environment:
    - OLLAMA_HOST=ollama:11434
  command:
    - "-c"
    - "sleep 3; ollama pull llama3.2"

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    networks: ['demo']
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - mysql_storage:/var/lib/mysql
    ports:
      - "3307:3306"  # Mapea el puerto interno 3306 a 3307 en el host para conexiones externas

  n8n-import:
    <<: *service-n8n
    hostname: n8n-import
    container_name: n8n-import
    entrypoint: /bin/sh
    command:
      - "-c"
      - "n8n import:credentials --separate --input=/backup/credentials && n8n import:workflow --separate --input=/backup/workflows"
    volumes:
      - ./n8n/backup:/backup
    depends_on:
      mysql:
        condition: service_healthy

  n8n:
    <<: *service-n8n
    hostname: n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - 5678:5678
    volumes:
      - n8n_storage:/home/node/.n8n
      - ./n8n/backup:/backup
      - ./shared:/data/shared
    depends_on:
      mysql:
        condition: service_healthy
      n8n-import:
        condition: service_completed_successfully

  qdrant:
    image: qdrant/qdrant
    hostname: qdrant
    container_name: qdrant
    networks: ['demo']
    restart: unless-stopped
    ports:
      - 6333:6333
    volumes:
      - qdrant_storage:/qdrant/storage

  ollama-cpu:
    profiles: ["cpu"]
    <<: *service-ollama

  ollama-gpu:
    profiles: ["gpu-nvidia"]
    <<: *service-ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  ollama-gpu-amd:
    profiles: ["gpu-amd"]
    <<: *service-ollama
    image: ollama/ollama:rocm
    devices:
      - "/dev/kfd"
      - "/dev/dri"

  ollama-pull-llama-cpu:
    profiles: ["cpu"]
    <<: *init-ollama
    depends_on:
      - ollama-cpu

  ollama-pull-llama-gpu:
    profiles: ["gpu-nvidia"]
    <<: *init-ollama
    depends_on:
      - ollama-gpu

  ollama-pull-llama-gpu-amd:
    profiles: [gpu-amd]
    <<: *init-ollama
    image: ollama/ollama:rocm
    depends_on:
     - ollama-gpu-amd
```

## Levantar el Entorno

Desde el directorio donde se encuentra tu docker-compose.yml, ejecuta:

```bash
docker-compose up -d
```

# 2. Configuración de la Base de Datos MySQL

Conectar desde TablePlus o MySQL Workbench

Utiliza los siguientes parámetros para conectarte al contenedor MySQL:

Host: localhost
Puerto: 3307
Usuario: el valor de MYSQL_USER (por ejemplo, n8n_user)
Contraseña: el valor de MYSQL_PASSWORD (por ejemplo, super-secret-pass)
Base de Datos: el valor de MYSQL_DATABASE (por ejemplo, n8n)

# Crear la Tabla y Funciones para Embeddings
Como MySQL no dispone de un tipo vector nativo, almacenaremos los embeddings como JSON y definiremos funciones almacenadas para calcular la similitud coseno.

## 2.1 Crear la Tabla documents

Ejecuta la siguiente consulta SQL en tu cliente MySQL:

```sql
CREATE TABLE IF NOT EXISTS documents (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  content TEXT,            -- Corresponde al contenido del documento
  metadata JSON,           -- Corresponde a metadatos del documento
  embedding JSON           -- Almacena el vector de embeddings (por ejemplo, un arreglo JSON de 1536 números)
);
```

## 2.2 Definir la Función para Calcular la Similitud Coseno

Esta función recorre los elementos de dos arreglos JSON y calcula el coseno de la similitud. Nota: Este proceso realiza 1536 iteraciones por cada cálculo y puede no ser óptimo para producción.

```sql
DELIMITER $$

CREATE FUNCTION vector_similarity(vec1 JSON, vec2 JSON) RETURNS DOUBLE
DETERMINISTIC
BEGIN
    DECLARE i INT DEFAULT 0;
    DECLARE n INT DEFAULT JSON_LENGTH(vec1);
    DECLARE dot DOUBLE DEFAULT 0;
    DECLARE norm1 DOUBLE DEFAULT 0;
    DECLARE norm2 DOUBLE DEFAULT 0;
    DECLARE v1 DOUBLE;
    DECLARE v2 DOUBLE;
    
    WHILE i < n DO
        SET v1 = JSON_EXTRACT(vec1, CONCAT('$[', i, ']'));
        SET v2 = JSON_EXTRACT(vec2, CONCAT('$[', i, ']'));
        SET dot = dot + (v1 * v2);
        SET norm1 = norm1 + (v1 * v1);
        SET norm2 = norm2 + (v2 * v2);
        SET i = i + 1;
    END WHILE;
    
    IF norm1 = 0 OR norm2 = 0 THEN
        RETURN 0;
    ELSE
        RETURN dot / (SQRT(norm1) * SQRT(norm2));
    END IF;
END$$

DELIMITER ;
```

## 2.3 Crear un Procedimiento para Buscar Documentos por Similitud

El siguiente procedimiento selecciona los documentos cuyo campo metadata cumple un filtro (en formato JSON) y los ordena de mayor a menor similitud en comparación con un vector de consulta dado. La similitud se calcula usando la función anterior.

```sql
DELIMITER $$

CREATE PROCEDURE match_documents (
    IN query_embedding JSON,
    IN match_count INT,
    IN filter JSON
)
BEGIN
    SELECT
        id,
        content,
        metadata,
        vector_similarity(embedding, query_embedding) AS similarity
    FROM documents
    WHERE JSON_CONTAINS(metadata, filter)
    ORDER BY vector_similarity(embedding, query_embedding) DESC
    LIMIT match_count;
END$$

DELIMITER ;
```