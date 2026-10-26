# Arquitectura y configuración

[← Inicio](../README.md) · [Instalación](instalacion.md) · [Uso y mantenimiento](uso.md)

Un único Gateway de OpenClaw gestiona los tres agentes. Cada uno consulta
un servidor MCP de haiku-rag que abre su base en modo lectura. La ingesta
corresponde a un único proceso compartido.

- [Servicios y versiones](#cómo-encajan-los-servicios)
- [Almacenamiento](#dónde-se-guardan-los-datos)
- [Archivos de configuración](#archivos-de-configuración)
- [Variables de entorno](#variables-de-entorno)
- [Permisos de los agentes](#permisos-de-los-agentes)
- [Referencias](#referencias)

## Dónde se guardan los datos

Todas las rutas son relativas al repositorio:

| Ruta | Contenido |
| --- | --- |
| `documents/` | Documentos originales; el ingester no los modifica |
| `data/openclaw/` | Configuración, sesiones y workspaces de los agentes |
| `data/auth/` | Directorio de secretos de perfiles de OpenClaw |
| `rag/<rol>/haiku.rag.lancedb/` | Base de cada ámbito |
| `rag/ingester/` | Cola y cachés del ingester |
| `rag/models/` | Modelos de Ollama, reranker y cachés de Docling |

`data/`, `rag/`, `documents/` y `.env` están excluidos de Git. Inclúyelos en tus
copias de seguridad. La guía de [mantenimiento](uso.md#copias-de-seguridad)
describe cómo detener los servicios para hacer una copia coherente.

## Cómo encajan los servicios

| Servicio | Función |
| --- | --- |
| OpenClaw `2026.10.7` | Gateway y panel web de Personal, Universidad y Técnico-Homelab |
| haiku-ingester `0.101.0` | Un único proceso que mantiene las tres bases |
| Docling Serve CPU `1.38.0` | Conversión y fragmentación mediante su API interna |
| Ollama `0.40.11` | Un `bge-m3` compartido para ingesta y consultas, con vectores de 1024 dimensiones |
| Infinity `0.0.77` | Un `BAAI/bge-reranker-v2-m3` compartido para ordenar resultados de búsqueda |
| Tres MCP haiku-rag `0.101.0` | Cada uno monta su base y la abre en solo lectura |

Ingester y MCP usan la imagen publicada `haiku.rag-slim`. El cliente
`provider: vllm` de haiku-rag consume la API compatible `/v1/rerank` de
Infinity. El modelo de reranking se carga únicamente en Infinity.

El embedder de [`config/ingester.yaml`](../config/ingester.yaml) y
[`config/haiku.rag.yaml`](../config/haiku.rag.yaml) debe coincidir.
Cambiar de modelo requiere reindexar los documentos, aunque conserve la dimensión.

La conversión documental corre en CPU. Ollama e Infinity usan la GPU NVIDIA
del host; sus modelos comparten esos recursos. Compose limita Ollama a un
modelo cargado y una petición en paralelo. Si también lo utilizas para el
chat, el modelo de conversación comparte ese servicio con los embeddings.

El modelo de conversación y sus credenciales se guardan en el estado de
OpenClaw; los parches del repositorio conservan esa elección.

## Archivos de configuración

| Archivo | Responsabilidad |
| --- | --- |
| [`compose.yaml`](../compose.yaml) | Servicios, versiones, montajes, GPU, red y comprobaciones de salud. |
| [`.env.example`](../.env.example) | Valores de partida para crear el `.env` local. |
| [`config/openclaw-agents.json`](../config/openclaw-agents.json) | Identidad y workspace de los tres agentes; conserva `main` como Personal. |
| [`config/openclaw-mcp.json`](../config/openclaw-mcp.json) | Registro de las tres conexiones MCP, inicialmente desactivadas, y filtro de lectura. |
| [`config/openclaw-permissions.json`](../config/openclaw-permissions.json) | Herramientas permitidas por agente y activación de los tres MCP. |
| [`config/agents/*/SOUL.md`](../config/agents/) | Roles, consulta de fuentes y pautas de respuesta. |
| [`config/ingester.yaml`](../config/ingester.yaml) | Bases, carpetas de origen, embeddings, conversión y cola de ingesta. |
| [`config/haiku.rag.yaml`](../config/haiku.rag.yaml) | Configuración compartida de embeddings y reranking para los MCP. |

Los parches se aplican en orden: **agentes → MCP → permisos**, siempre con
`--dry-run` primero. Se combinan con la configuración existente. El parche de
permisos reemplaza las listas de herramientas de los tres agentes definidos.
Consulta los comandos completos en la [guía de instalación](instalacion.md#5-configurar-los-agentes-y-sus-roles).

## Variables de entorno

| Variable | Valor por defecto | Uso |
| --- | --- | --- |
| `OPENCLAW_PORT` | `18789` | Puerto del panel en `127.0.0.1`; el puerto interno sigue siendo `18789`. |
| `TZ` | `Europe/Madrid` | Zona horaria del contenedor OpenClaw. |
| `DOCLING_MEMORY_LIMIT` | `4g` | Límite de RAM de Docling; no reserva esa memoria. |

Las imágenes de los servicios están fijadas en Compose. `OLLAMA_IMAGE` solo
sobrescribe la imagen del servicio auxiliar `ollama-pull`.

Si cambias `OPENCLAW_PORT`, ajusta también los orígenes permitidos del Gateway
y el acceso al panel según la [guía de instalación](instalacion.md#3-configurar-openclaw-y-arrancar).

## Permisos de los agentes

| Agente | ID | Prefijo de herramientas RAG | Workspace en `data/openclaw/` |
| --- | --- | --- | --- |
| Personal | `main` | `rag-personal__` | `workspace/` |
| Universidad | `universidad` | `rag-universidad__` | `workspace-universidad/` |
| Técnico-Homelab | `tecnico-homelab` | `rag-tecnico-homelab__` | `workspace-tecnico-homelab/` |

Cada conexión expone cinco herramientas de lectura: `search_documents`,
`list_documents`, `get_document`, `get_document_outline` y `get_document_section`.
Los agentes pueden mantener archivos en su propio workspace. Sus listas de
herramientas excluyen ejecución de comandos, sesiones, navegación web, control
del Gateway y búsqueda de memoria con rutas adicionales. `execute_code` queda
fuera del filtro MCP; el socket Docker no se monta en OpenClaw.

Los roles indican cuándo consultar y cómo citar las fuentes. La configuración
de herramientas limita a qué puede acceder cada agente. Son permisos dentro
de un mismo proceso de Gateway; no separan usuarios del sistema operativo
ni constituyen entornos de ejecución independientes.

El panel es el único puerto publicado y escucha en la interfaz de loopback.
Los MCP, Docling, Ollama e Infinity se comunican por la red interna de Compose.
El ingester monta los originales en solo lectura y conserva el contenido
indexado cuando desaparece un original (`delete_orphans: false`).

## Referencias

Documentación de los proyectos que componen el despliegue. Los enlaces
versionados corresponden a las versiones fijadas en Compose.

- [Docker de OpenClaw](https://docs.openclaw.ai/install/docker)
- [Asistente de OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/cli/onboard.md)
- [Acceso al panel en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/cli/dashboard.md)
- [Agentes en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/cli/agents.md)
- [Parches de configuración en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/cli/config.md#config-patch)
- [Conexiones MCP en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/tools/mcp.md)
- [Permisos de herramientas en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/gateway/config-tools/tool-policy.md)
- [Herramientas MCP de haiku-rag 0.101.0](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/mcp.md)
- [Workspaces y archivos de rol en OpenClaw 2026.10.7](https://github.com/openclaw/openclaw/blob/v2026.10.7/docs/concepts/agent-workspace.md)
- [Imágenes publicadas de haiku-rag](https://ggozad.github.io/haiku.rag/installation/#docker)
- [Docling remoto y compatibilidad](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/remote-processing.md)
- [Ingesta multibase](https://github.com/ggozad/haiku.rag/blob/0.101.0/docs/ingester.md)
- [PDF grandes y split_pages](https://ggozad.github.io/haiku.rag/configuration/processing/#large-pdfs-and-docling-memory)
- [BGE-M3 en Ollama](https://ollama.com/library/bge-m3)
- [Especificaciones de BGE-M3](https://huggingface.co/BAAI/bge-m3)
- [Infinity](https://github.com/michaelfeil/infinity/tree/0.0.77)
