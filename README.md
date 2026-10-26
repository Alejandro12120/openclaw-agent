# OpenClaw + haiku-rag

Infraestructura para un OpenClaw y tres bases RAG: **personal**, **universidad**
y **técnico-homelab**. Comparten la ingesta, los embeddings y el reranker.

Cada agente consulta su RAG y mantiene sus notas en su propio workspace.
Los permisos bloquean comandos, sesiones ajenas y herramientas de otros RAG.

**Primera instalación:** sigue los pasos de [puesta en marcha](#puesta-en-marcha).
Si OpenClaw ya está configurado, ve a [abrir el panel](#4-abrir-el-panel) y
[configurar los agentes](#5-configurar-los-agentes-y-sus-roles); después sigue
los pasos 6 y 7 para conectar sus RAG y aplicar los permisos.

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
mkdir -p data/openclaw/workspace data/openclaw/workspace-universidad
mkdir -p data/openclaw/workspace-tecnico-homelab
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

### 5. Configurar los agentes y sus roles

Tras configurar el proveedor del modelo en el paso 3, aplica
[`config/openclaw-agents.json`](config/openclaw-agents.json). Conserva el agente
actual con ID `main` como **Personal** y añade los otros dos:

| ID | Nombre y función | Workspace dentro de `data/openclaw/` |
| --- | --- | --- |
| `main` | **Personal**: organización, tareas y planes personales; agente por defecto | `workspace/` |
| `universidad` | **Universidad**: tutoría, apuntes, ejercicios y estudio | `workspace-universidad/` |
| `tecnico-homelab` | **Técnico-Homelab**: sistemas, redes, contenedores y diagnóstico | `workspace-tecnico-homelab/` |

Los roles están en los archivos `SOUL.md` de [`config/agents/`](config/agents/).
Compose los monta en solo lectura: edita esos archivos y repite la recreación
de OpenClaw indicada abajo para cambiar los roles.
Si ya personalizaste el `SOUL.md` del agente actual, incorpora lo que quieras
conservar al rol Personal antes de continuar; el montaje oculta el archivo
anterior, aunque lo conserva en `data/`.

En instalaciones existentes, crea también los tres directorios de workspace
del paso 2 para que pertenezcan a tu usuario, antes de ejecutar estos comandos.
Con OpenClaw arrancado como en el paso 3, valida primero el parche sin escribir
la configuración. Usa `exec` para ejecutarlo en el mismo contenedor del Gateway:

```sh
docker compose exec -T openclaw node \
  dist/index.js config patch --stdin --dry-run < config/openclaw-agents.json
```

Si la validación termina correctamente, aplica el mismo parche y recrea
OpenClaw para cargar los montajes de roles:

```sh
docker compose exec -T openclaw node \
  dist/index.js config patch --stdin < config/openclaw-agents.json
docker compose up -d --no-deps --force-recreate --wait openclaw
docker compose exec openclaw node dist/index.js agents list
```

El parche se combina con la configuración existente y se puede repetir:
conserva modelos, credenciales, sesiones y otros agentes que hayas añadido.
Los nuevos agentes heredan el modelo por defecto configurado en OpenClaw.
En el panel, abre **Agents** y usa **Open chat** en el agente que quieras.
Los canales y su enrutamiento quedan pendientes de configurar.

Los roles son instrucciones de comportamiento. Los permisos se aplican por
separado en el paso 7; crear tres agentes y tres bases no los configura.

Los tres roles exigen consultar su RAG antes de responder preguntas de contenido
y leer los archivos de su propio workspace cuando la pregunta se refiera a ellos.
Deben citar las fuentes consultadas y avisar si faltan datos o falla la conexión.
Los saludos, las aclaraciones y las transformaciones de texto aportado que no
añadan hechos externos quedan exentos de buscar.

Si solo cambias los `SOUL.md`, recrea OpenClaw con el comando anterior y abre
un chat nuevo de cada agente; no hace falta reaplicar los parches. Para comprobar
la conducta, pregunta por un documento conocido y revisa que haya una llamada
real a `rag-…__search_documents` antes de la respuesta, con una cita pertinente.
Prueba también una pregunta sin respaldo documental: debe indicar qué falta.

### 6. Registrar las conexiones MCP

[`config/openclaw-mcp.json`](config/openclaw-mcp.json) conecta OpenClaw con los
tres servicios haiku-rag mediante Streamable HTTP, sin publicar más puertos
ni instalar clientes adicionales. Cada conexión ofrece cinco herramientas:
buscar, listar, leer un documento, consultar su índice y leer una sección.
El filtro excluye `execute_code`, que ejecuta Python en el servidor RAG.

Con el Gateway arrancado, valida y aplica el parche:

```sh
docker compose exec -T openclaw node \
  dist/index.js config patch --stdin --dry-run < config/openclaw-mcp.json
docker compose exec -T openclaw node \
  dist/index.js config patch --stdin < config/openclaw-mcp.json
docker compose exec openclaw node dist/index.js mcp status --verbose
```

El parche conserva el resto de la configuración, incluidos otros servidores
MCP. Las entradas `rag-personal`, `rag-universidad` y `rag-tecnico-homelab`
deben aparecer desactivadas. El paso 7 las activa al aplicar los permisos.
Reaplicar este parche vuelve a desactivarlas; repite después el paso 7.

### 7. Aplicar los permisos y activar los RAG

Aplica [`config/openclaw-permissions.json`](config/openclaw-permissions.json)
después de los dos parches anteriores. La lista de herramientas permitidas es
específica para cada agente:

| Agente | RAG disponible | Notas |
| --- | --- | --- |
| Personal (`main`) | `rag-personal` | Solo su workspace |
| Universidad | `rag-universidad` | Solo su workspace |
| Técnico-Homelab | `rag-tecnico-homelab` | Solo su workspace |

Los tres pueden buscar y leer documentos de su base y mantener sus notas.
No pueden ejecutar comandos, usar sesiones de otros agentes, navegar por la web
ni cambiar la configuración del Gateway. Tampoco reciben `execute_code` del RAG.
La ingesta sigue siendo responsabilidad del único `haiku-ingester`.

```sh
docker compose exec -T openclaw node \
  dist/index.js config patch --stdin --dry-run < config/openclaw-permissions.json
docker compose exec -T openclaw node \
  dist/index.js config patch --stdin < config/openclaw-permissions.json
docker compose up -d --no-deps --force-recreate --wait openclaw
docker compose exec openclaw node dist/index.js config validate
docker compose exec openclaw node dist/index.js mcp probe --json
```

`mcp probe` debe mostrar los tres servidores, cinco herramientas por servidor
y `diagnostics` vacío. Es una comprobación del Gateway: no representa el acceso
de un agente. En el panel, abre un chat nuevo de cada agente y revisa su lista
de herramientas: solo debe ofrecer su prefijo `rag-…__` y archivos del workspace.
Pídele que consulte un documento conocido de su RAG y que intente consultar
uno de otro ámbito. La segunda consulta debe quedar fuera de sus herramientas.

El parche reemplaza las listas de herramientas de estos tres agentes y conserva
modelos, credenciales, sesiones y agentes adicionales. Las restricciones globales
o del proveedor pueden limitar todavía más las herramientas. Si el `--dry-run`
detecta un `tools.alsoAllow` personalizado incompatible con `tools.allow`, revisa
esa personalización antes de aplicar el cambio.

Las notas se leen y escriben como archivos del workspace. No se habilitan las
herramientas de búsqueda de memoria, que pueden tener rutas adicionales
configuradas fuera de él.

Estos son permisos de herramientas dentro de un único Gateway, que mantiene las
conexiones a los tres MCP. No separan procesos ni usuarios del sistema operativo.
No se añade acceso Docker ni ejecución de comandos; un sandbox de ejecución
queda para cuando sea necesario.

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

### Error al inicializar un workspace

Si el chat muestra `Workspace filesystem does not support atomic bootstrap
publication`, comprueba primero el propietario del workspace. OpenClaw
`2026.10.7` también muestra ese mensaje cuando no puede escribir por falta de
permisos, aunque el disco admita enlaces duros.

Docker puede crear los directorios de Universidad y Técnico-Homelab como `root`
si no existían al montar sus roles. Comprueba su propietario dentro del contenedor:

```sh
docker compose exec openclaw ls -ldn \
  /home/node/.openclaw/workspace \
  /home/node/.openclaw/workspace-universidad \
  /home/node/.openclaw/workspace-tecnico-homelab
```

Si los dos nuevos workspaces pertenecen a `0:0`, ajusta solo esos directorios
al usuario `1000:1000` de OpenClaw y vuelve a enviar el mensaje:

```sh
docker compose exec --user 0:0 openclaw chown 1000:1000 \
  /home/node/.openclaw/workspace-universidad \
  /home/node/.openclaw/workspace-tecnico-homelab
```

El cambio se aplica sin reiniciar. Los archivos `SOUL.md` siguen montados en
solo lectura. Para evitarlo en instalaciones nuevas, crea los directorios del
paso 2 con tu usuario antes de arrancar Compose.

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
copias de seguridad.

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

El embedder de `config/ingester.yaml` y `config/haiku.rag.yaml` debe coincidir.
Cambiar de modelo requiere reindexar los documentos, aunque conserve la dimensión.

### Red y próximos pasos

Quedan pendientes los canales y su enrutamiento. OpenClaw comparte red con los
tres MCP; el acceso de cada agente se filtra en sus herramientas. Los endpoints
internos son
`http://haiku-rag-personal:8001/mcp`, `http://haiku-rag-universidad:8001/mcp` y
`http://haiku-rag-tecnico-homelab:8001/mcp`.

## Referencias

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
