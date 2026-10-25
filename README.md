# OpenClaw + haiku-rag

Un OpenClaw y tres servidores MCP, con bases independientes para `personal`,
`universidad` y `tecnico-homelab`. Un único ingester mantiene las tres bases.

## Documentos y datos

Los originales están separados del estado de la aplicación:

- `documents/personal/` alimenta la base personal.
- `documents/universidad/` alimenta la base de universidad.
- `documents/tecnico/` alimenta la base técnico-homelab.

El ingester monta `./documents` en `/documents` **en solo lectura**. Escribe las
bases en `rag/<rol>/haiku.rag.lancedb/` y la cola y sus cachés en `rag/ingester/`.
Cada MCP monta únicamente la carpeta de su propia base.

OpenClaw guarda configuración, sesiones y workspace en `data/openclaw/`, y su
directorio de secretos de perfiles en `data/auth/`. Ollama e Infinity conservan
sus modelos en `rag/models/ollama/` y `rag/models/reranker/`. Docling conserva sus
cachés adicionales en `rag/models/docling/`.

`data/`, `rag/`, `documents/` y `.env` están excluidos de Git. Inclúyelos en las
copias de seguridad. Borrar un original conserva su contenido indexado
(`delete_orphans: false`).

## Servicios y modelos

- **Ingester:** imagen publicada haiku.rag-slim `0.101.0`, con un único proceso
  que vigila las tres carpetas y escribe en sus bases correspondientes.
- **Docling:** imagen publicada docling-serve-cpu `1.38.0`, compatible con esta
  versión de haiku-rag. Convierte y fragmenta documentos mediante su API interna.
- **Embeddings:** un Ollama `0.40.11` sirve `bge-m3` para la ingesta y las consultas
  de las tres bases. Los vectores tienen 1024 dimensiones. `ollama-pull` descarga
  el modelo antes de que arranque el ingester.
- **Reranking:** un Infinity `0.0.77` sirve `BAAI/bge-reranker-v2-m3` a los tres MCP.
  Se ejecuta al buscar. El cliente `provider: vllm` de haiku-rag consume su API
  compatible `/v1/rerank`; el modelo está cargado únicamente en Infinity.
- **Lectores:** los tres MCP usan haiku.rag-slim `0.101.0` y abren las bases en modo
  de solo lectura. OpenClaw usa la imagen oficial `2026.10.7`.

Ollama e Infinity usan NVIDIA. La elección del LLM de OpenClaw es independiente.
El embedder de `config/ingester.yaml` y `config/haiku.rag.yaml` debe coincidir.
Cambiar de modelo requiere reindexar los documentos, aunque conserve la dimensión.

## Documentos grandes

`config/ingester.yaml` configura `processing.split_pages: 10`:
los PDF se convierten por bloques de diez páginas y después se unen conservando
la numeración. Un documento de 21 páginas se procesa en bloques de 10, 10 y 1.
La configuración usa un único worker y omite las imágenes de páginas y figuras;
la ingesta conserva el texto y la estructura sin necesitar un modelo de visión.
`DOCLING_MEMORY_LIMIT` limita la RAM de Docling; su valor inicial es `4g`.
Ajústalo al tamaño y complejidad de tus documentos. El troceado reduce el pico
de conversión, pero no garantiza que cualquier PDF quepa en ese límite.

## Preparación

Requiere Linux x86_64, Docker Engine, Docker Compose 2.30 o superior y
[NVIDIA Container Toolkit configurado para Docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).
El repositorio no instala paquetes ni modifica la configuración del host.
Todos los servicios usan imágenes publicadas; no hay compilaciones locales.
La versión mínima de Compose permite usar `gpus: all` en Ollama e Infinity.

```sh
cp .env.example .env
mkdir -p documents/personal documents/universidad documents/tecnico
mkdir -p data/openclaw data/auth rag/ingester
mkdir -p rag/personal rag/universidad rag/tecnico-homelab
mkdir -p rag/models/ollama rag/models/reranker rag/models/docling
chmod 700 data/openclaw data/auth rag rag/ingester rag/models
docker compose config --quiet
docker compose pull
```

Las versiones predeterminadas de las imágenes se fijan en `compose.yaml`.
Para sustituir una imagen, descomenta su variable `*_IMAGE` en `.env` y asigna
la imagen y etiqueta deseadas. Si ya tienes un `.env` con estas variables,
sus valores prevalecen; coméntalas o elimínalas para usar los predeterminados.

Los servicios de datos escriben con UID/GID `1000:1000`. Ese usuario debe poder
escribir en `data/` y `rag/`, y leer los originales de `documents/`.
Comprueba también el acceso a la GPU:

```sh
docker compose run --rm --no-deps --entrypoint nvidia-smi ollama
```

## Primer arranque

Configura OpenClaw con su asistente interactivo. Las credenciales se introducen
localmente y se guardan en `data/`:

```sh
docker compose run --rm --no-deps --entrypoint node openclaw \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw \
  dist/index.js config set --batch-json '[
    {"path":"gateway.mode","value":"local"},
    {"path":"gateway.bind","value":"lan"},
    {"path":"gateway.controlUi.allowedOrigins","value":[
      "http://localhost:18789","http://127.0.0.1:18789"
    ]}
  ]'
docker compose up -d --wait --wait-timeout 1200
docker compose ps
```

El ingester inicializa las tres bases y los MCP esperan a su healthcheck.
Las primeras descargas de imágenes y modelos pueden tardar varios minutos.
Docling incluye sus modelos de conversión y cachea los tokenizadores adicionales.

Abre OpenClaw en <http://127.0.0.1:18789>. Si cambias `OPENCLAW_PORT`, adapta también
los puertos de `allowedOrigins`. Los servicios RAG y de modelos no publican puertos
en el host; la API de control del ingester escucha en su propio contenedor.

## Uso

Coloca los archivos en la subcarpeta correspondiente de `documents/`. El ingester
detecta cambios y hace barridos periódicos. Para revisar la ingesta y listar una base:

```sh
docker compose logs --tail 100 haiku-ingester
docker compose exec haiku-rag-personal haiku-rag --config /app/haiku.rag.yaml list
```

Los lectores pueden tardar hasta 30 segundos en ver nuevas escrituras. No arranques
otro ingester ni comandos de escritura contra estas bases mientras el servicio
esté funcionando. Para parar conservando los datos: `docker compose down`.
Para arrancar de nuevo: `docker compose up -d`.

Quedan pendientes los agentes, sus roles, la integración MCP y los permisos por
agente. OpenClaw todavía comparte red con los tres MCP; aún no está restringido
a qué base puede acceder cada agente. Sus endpoints internos son
`http://haiku-rag-personal:8001/mcp`, `http://haiku-rag-universidad:8001/mcp` y
`http://haiku-rag-tecnico-homelab:8001/mcp`.

## Referencias

- [Docker de OpenClaw](https://docs.openclaw.ai/install/docker)
- [Imágenes publicadas de haiku-rag](https://ggozad.github.io/haiku.rag/installation/#docker)
- [Docling remoto y compatibilidad](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/remote-processing.md)
- [Ingesta multibase](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/ingester.md)
- [PDF grandes y split_pages](https://ggozad.github.io/haiku.rag/configuration/processing/#large-pdfs-and-docling-memory)
- [BGE-M3 en Ollama](https://ollama.com/library/bge-m3)
- [Especificaciones de BGE-M3](https://huggingface.co/BAAI/bge-m3)
- [Infinity](https://github.com/michaelfeil/infinity/tree/0.0.77)
