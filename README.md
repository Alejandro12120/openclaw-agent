# OpenClaw + haiku-rag

Infraestructura para un OpenClaw y tres bases RAG: **personal**, **universidad**
y **técnico-homelab**. Comparten la ingesta, los embeddings y el reranker.

Esta fase prepara los servicios. La creación de los tres agentes, la conexión
con sus RAG y los permisos por agente quedan para las siguientes PR.

**Primera instalación:** sigue los pasos de [puesta en marcha](#puesta-en-marcha).
Si OpenClaw ya está configurado, ve directamente a [abrir el panel](#4-abrir-el-panel).

## Puesta en marcha

Ejecuta los comandos desde la raíz del repositorio, en el servidor.

### 1. Comprobar los requisitos

Necesitas Linux x86_64, Docker Engine, Compose **2.30 o superior** y una GPU
NVIDIA con su driver y el
[NVIDIA Container Toolkit configurado para Docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-docker).
Ollama e Infinity usan la GPU; Docling trabaja en CPU.

```sh
docker compose version
nvidia-smi
```

El repositorio no instala paquetes ni reconfigura el host.

### 2. Preparar las carpetas

En una instalación nueva, copia la configuración de ejemplo y crea los
directorios. Si ya tienes `.env`, conserva el tuyo.

```sh
test -e .env || cp .env.example .env
mkdir -p documents/personal documents/universidad documents/tecnico
mkdir -p data/openclaw data/auth rag/ingester
mkdir -p rag/personal rag/universidad rag/tecnico-homelab
mkdir -p rag/models/ollama rag/models/reranker rag/models/docling
chmod 700 data/openclaw data/auth rag rag/ingester rag/models
```

Los contenedores escriben con UID/GID `1000:1000`. Ese usuario debe poder
escribir en `data/` y `rag/`, y leer `documents/`. Si tu usuario tiene otro UID,
ajusta el propietario de estas carpetas antes de continuar.

Puedes editar `.env` para cambiar el puerto del panel, la zona horaria o el
límite de RAM de Docling. Después valida el Compose y descarga las imágenes:

```sh
docker compose config --quiet
docker compose pull
docker compose run --rm --no-deps --entrypoint nvidia-smi ollama
```

La última comprobación debe mostrar la GPU desde el contenedor. Que funcione
`nvidia-smi` en el host no basta: Docker también necesita la integración NVIDIA.
Si aparece `failed to discover GPU vendor from CDI`, revisa el Toolkit antes
de continuar.

Las versiones están fijadas en `compose.yaml`; todos los servicios usan
imágenes publicadas, sin compilaciones locales.

### 3. Configurar OpenClaw y arrancar

OpenClaw necesita dos autenticaciones distintas:

- **Proveedor del modelo:** su API key o inicio de sesión, según el proveedor.
  El asistente permite configurarlo. `bge-m3` sirve para embeddings; el modelo
  de conversación de OpenClaw se elige por separado.
- **Panel web:** el asistente genera un token del Gateway automáticamente.
  Es normal que no te pida una contraseña para el panel. En el paso siguiente
  obtendrás el enlace de acceso sin buscar ni copiar ese token.

Abre el asistente en una terminal interactiva:

```sh
docker compose run --rm -it --no-deps --entrypoint node openclaw \
  dist/index.js onboard --flow advanced --mode local --gateway-auth token \
  --no-install-daemon --skip-health --skip-ui
```

Selecciona el proveedor, completa su autenticación y elige el modelo. Si
aparecen valores existentes, puedes conservarlos o modificarlos; no necesitas
reiniciar la configuración. Si omites el proveedor, podrás abrir el panel,
pero el chat quedará pendiente de configurar.

En las opciones del Gateway, usa el puerto interno **18789**, el bind **LAN**
y Tailscale **Off**. El servicio lo ejecuta Docker; puedes omitir canales,
skills e integraciones en esta fase. El asistente guarda el estado en `data/`.

A continuación, aplica la configuración de red para este Compose:

```sh
docker compose run --rm --no-deps --entrypoint node openclaw \
  dist/index.js config set --batch-json '[
    {"path":"gateway.mode","value":"local"},
    {"path":"gateway.bind","value":"lan"},
    {"path":"gateway.controlUi.allowedOrigins","value":[
      "http://localhost:18789","http://127.0.0.1:18789"
    ]}
  ]'
```

Si cambiaste `OPENCLAW_PORT` en `.env`, sustituye `18789` en las dos URL de
`allowedOrigins` por ese puerto. El puerto interno del Gateway sigue siendo
`18789`.

Arranca los servicios:

```sh
docker compose up -d --wait --wait-timeout 1200
docker compose ps
```

El primer arranque puede tardar varios minutos: se descargan `bge-m3` y el
reranker, el ingester inicializa las bases y los MCP esperan a que esté listo.
Docling incluye sus modelos de conversión y cachea tokenizadores adicionales.
`ollama-pull` termina con código `0` al completar la descarga; es lo esperado.

### 4. Abrir el panel

Con OpenClaw arrancado, obtén un enlace de acceso desde el host:

```sh
docker compose exec openclaw node dist/index.js dashboard --json
```

**Abre en tu navegador la dirección del campo `browserUrl`**, sin las comillas.
Es un enlace de emparejamiento de un solo uso que caduca a los diez minutos.
Si caduca o usas otro navegador, ejecuta de nuevo el comando. Tras emparejarlo,
ese navegador conserva su acceso al panel.

En OpenClaw `2026.10.7`, `dashboard --no-open` solo imprime la URL base en una
terminal sin portapapeles: no proporciona el enlace autenticado. Usa `--json`
y `browserUrl` también si el panel muestra `unauthorized` o pide emparejarlo.
No hace falta entrar en una shell del contenedor ni leer `openclaw.json`.

Si cambiaste `OPENCLAW_PORT`, ajusta el puerto tanto de la dirección HTTP como
del parámetro `gatewayUrl` del enlace. Conserva los valores de emparejamiento.

**Si el navegador está en otro equipo**, abre primero un túnel desde ese equipo
(sustituye `usuario@servidor` por tu conexión SSH):

```sh
ssh -N -L 18789:127.0.0.1:18789 usuario@servidor
```

Mantén el túnel abierto y abre allí el enlace. Si cambiaste `OPENCLAW_PORT`,
sustituye solo el puerto de destino (el último `18789`) en el comando SSH;
así puedes abrir el enlace original sin modificarlo. El panel se publica solo
en `127.0.0.1`; los servicios RAG y de modelos no publican puertos en el host.

## Añadir documentos

Coloca cada archivo en la carpeta de su ámbito:

| Carpeta de originales | Base RAG |
| --- | --- |
| `documents/personal/` | `personal` |
| `documents/universidad/` | `universidad` |
| `documents/tecnico/` | `tecnico-homelab` |

El ingester lee `./documents` **en solo lectura**, detecta cambios y realiza
barridos periódicos. Para comprobar la ingesta y listar la base personal:

```sh
docker compose logs --tail 100 haiku-ingester
docker compose exec haiku-rag-personal haiku-rag --config /app/haiku.rag.yaml list
```

Los MCP pueden tardar hasta 30 segundos en ver los cambios. Si borras un
original, su contenido indexado se conserva (`delete_orphans: false`).
Mantén un único escritor: no ejecutes otro ingester ni comandos de escritura
contra estas bases mientras el servicio esté funcionando.

### PDF grandes

En `config/ingester.yaml`, `processing.split_pages: 10` convierte los PDF por
bloques de diez páginas y une el resultado conservando la numeración. Por
ejemplo, un PDF de 21 páginas se procesa en bloques de 10, 10 y 1.

La ingesta usa un worker, conserva texto y estructura y omite imágenes de
páginas y figuras. Docling tiene un límite inicial de **4 GB de RAM**,
ajustable con `DOCLING_MEMORY_LIMIT` en `.env`. El troceado reduce el pico de
conversión, aunque algunos documentos pueden necesitar un límite mayor.

## Operaciones habituales

```sh
docker compose ps                  # Estado de los servicios
docker compose logs --tail 100      # Últimos mensajes
docker compose down                # Parar conservando los datos
docker compose up -d                # Volver a arrancar
```

Para cambiar el proveedor o las credenciales del modelo, vuelve a ejecutar
el asistente del paso 3 y reinicia OpenClaw con `docker compose restart openclaw`.
Para recuperar el acceso al panel basta con el comando del paso 4.

## Dónde se guardan los datos

Todas las rutas son relativas al repositorio:

| Ruta | Contenido |
| --- | --- |
| `documents/` | Documentos originales; el ingester no los modifica |
| `data/openclaw/` | Configuración, sesiones y workspace de OpenClaw |
| `data/auth/` | Directorio de secretos de perfiles de OpenClaw |
| `rag/<rol>/haiku.rag.lancedb/` | Base de cada ámbito |
| `rag/ingester/` | Cola y cachés del ingester |
| `rag/models/` | Modelos de Ollama, reranker y cachés de Docling |

`data/`, `rag/`, `documents/` y `.env` están excluidos de Git. Inclúyelos en tus
copias de seguridad.

## Cómo encajan los servicios

| Servicio | Función |
| --- | --- |
| OpenClaw `2026.10.7` | Gateway y panel web de los futuros agentes |
| haiku-ingester `0.101.0` | Un único proceso que mantiene las tres bases |
| Docling Serve CPU `1.38.0` | Conversión y fragmentación mediante su API interna |
| Ollama `0.40.11` | Un `bge-m3` compartido para ingesta y consultas, con vectores de 1024 dimensiones |
| Infinity `0.0.77` | Un `BAAI/bge-reranker-v2-m3` compartido para ordenar resultados de búsqueda |
| Tres MCP haiku-rag `0.101.0` | Cada uno monta su base y la abre en solo lectura |

Ingester y MCP usan la imagen publicada `haiku.rag-slim`. El cliente
`provider: vllm` de haiku-rag consume la API compatible `/v1/rerank` de
Infinity. El modelo de reranking se carga únicamente en Infinity.

El embedder de `config/ingester.yaml` y `config/haiku.rag.yaml` debe coincidir.
Cambiar de modelo requiere reindexar los documentos, aunque conserve la dimensión.

### Pendiente para las siguientes PR

Quedan pendientes los agentes, sus roles, la integración MCP y los permisos por
agente. OpenClaw todavía comparte red con los tres MCP; aún no está restringido
a qué base puede acceder cada agente. Sus endpoints internos son
`http://haiku-rag-personal:8001/mcp`, `http://haiku-rag-universidad:8001/mcp` y
`http://haiku-rag-tecnico-homelab:8001/mcp`.

## Referencias

- [Docker de OpenClaw](https://docs.openclaw.ai/install/docker)
- [Asistente de OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/cli/onboard.md)
- [Acceso al panel en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/cli/dashboard.md)
- [Imágenes publicadas de haiku-rag](https://ggozad.github.io/haiku.rag/installation/#docker)
- [Docling remoto y compatibilidad](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/remote-processing.md)
- [Ingesta multibase](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/ingester.md)
- [PDF grandes y split_pages](https://ggozad.github.io/haiku.rag/configuration/processing/#large-pdfs-and-docling-memory)
- [BGE-M3 en Ollama](https://ollama.com/library/bge-m3)
- [Especificaciones de BGE-M3](https://huggingface.co/BAAI/bge-m3)
- [Infinity](https://github.com/michaelfeil/infinity/tree/0.0.77)
