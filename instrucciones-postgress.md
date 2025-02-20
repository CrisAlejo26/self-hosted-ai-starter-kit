# Instrucciones para Integrar pgvector en n8n con PostgreSQL

Este documento describe los pasos para configurar un entorno Docker con n8n y dos contenedores PostgreSQL (uno para n8n y otro para trabajar con embeddings utilizando la extensión pgvector). Además, se explica cómo ejecutar los scripts SQL necesarios en TablePlus.

---

## Requisitos Previos

- **Docker** y **Docker Compose** instalados en tu sistema.
- Un archivo `.env` con las siguientes variables definidas:
  ```env
  POSTGRES_USER=root
  POSTGRES_PASSWORD=password
  POSTGRES_DB=n8n
  N8N_ENCRYPTION_KEY=super-secret-key
  N8N_USER_MANAGEMENT_JWT_SECRET=even-more-secret


Paso 1: Configuración del Entorno Docker
Hemos actualizado el archivo docker-compose.yml para definir los siguientes servicios:

n8n: Plataforma low-code que se conecta a la base de datos PostgreSQL.
postgres: Contenedor para la base de datos de n8n.
postgressDb: Contenedor personalizado para PostgreSQL que incluye la extensión pgvector (utilizando la imagen my-postgres-vector).
Otras dependencias: Ollama, Qdrant, etc.

```yaml
volumes:
  n8n_storage:
  postgres_storage:
  ollama_storage:
  qdrant_storage:

networks:
  demo:

x-n8n: &service-n8n
  image: n8nio/n8n:latest
  networks: ['demo']
  environment:
    - DB_TYPE=postgresdb
    - DB_POSTGRESDB_HOST=postgres
    - DB_POSTGRESDB_USER=${POSTGRES_USER}
    - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
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
  postgres:
    image: postgres:16-alpine
    hostname: postgres
    networks: ['demo']
    restart: unless-stopped
    environment:
      - POSTGRES_USER
      - POSTGRES_PASSWORD
      - POSTGRES_DB
    volumes:
      - postgres_storage:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 5s
      timeout: 5s
      retries: 10

  postgressDb:
    image: my-postgres-vector
    hostname: postgres
    networks:
      - demo
    restart: unless-stopped
    environment:
      - POSTGRES_USER
      - POSTGRES_PASSWORD
      - POSTGRES_DB
    volumes:
      - postgres_storage:/var/lib/postgresql/data
    ports:
      - "3321:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

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
      postgres:
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
      postgres:
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

# Paso 2: Construir la Imagen Personalizada de PostgreSQL con pgvector
Si aún no cuentas con la imagen my-postgres-vector, puedes crearla siguiendo estos pasos:

Crear un archivo Dockerfile.pgvector:

```Dockerfile
    FROM postgres:16-alpine

# Instalar dependencias necesarias
RUN apk add --no-cache --update git make gcc musl-dev clang llvm-dev cmake

# Clonar y compilar pgvector (ajusta la versión según sea necesario)
RUN git clone --depth 1 --branch v0.5.0 https://github.com/pgvector/pgvector.git /tmp/pgvector \
    && cd /tmp/pgvector \
    && make \
    && make install \
    && rm -rf /tmp/pgvector
```

Construir la imagen:

Ejecuta el siguiente comando en el directorio donde se encuentre el Dockerfile:

```bash
docker build -f Dockerfile.pgvector -t my-postgres-vector .
```

# Paso 3: Levantar el Entorno con Docker Compose

En la terminal, navega al directorio donde se encuentra tu archivo docker-compose.yml.

Ejecuta:

```bash
docker-compose up -d
```

# Paso 4: Configurar la Base de Datos en TablePlus

Para agregar la funcionalidad de embeddings con pgvector, deberás ejecutar los siguientes scripts SQL en el contenedor postgressDb.

Conectar TablePlus
Host: localhost (o la IP de tu máquina)
Puerto: 3321 (ya que mapeamos el puerto 5432 interno a 3321)
Usuario: el valor de POSTGRES_USER (por ejemplo, root)
Contraseña: el valor de POSTGRES_PASSWORD (por ejemplo, password)
Base de Datos: el valor de POSTGRES_DB (por ejemplo, n8n)

# Ejecutar el Script SQL

Copia y ejecuta el siguiente script en TablePlus uno despues del otro:

```sql
-- 1. Habilitar la extensión pgvector (ya que la imagen la incluye)
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Crear la tabla 'documents' para almacenar documentos con embeddings
CREATE TABLE IF NOT EXISTS documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,          -- Contenido del documento
  metadata JSONB,        -- Metadatos asociados al documento
  embedding VECTOR(1536) -- Dimensión del vector (ajusta según el modelo de embeddings)
);

-- 3. Crear la función para buscar documentos por similitud
CREATE OR REPLACE FUNCTION match_documents (
  query_embedding VECTOR(1536),
  match_count INT DEFAULT NULL,
  filter JSONB DEFAULT '{}'
)
RETURNS TABLE (
  id BIGINT,
  content TEXT,
  metadata JSONB,
  similarity FLOAT
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE metadata @> filter
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```