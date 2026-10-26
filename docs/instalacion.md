# Instalación

[← Inicio](../README.md) · [Uso y mantenimiento](uso.md) · [Arquitectura](arquitectura.md)

Esta guía recorre la instalación completa: prepara los directorios, configura
el modelo de conversación y conecta cada agente con su base documental.

Ejecuta los comandos desde la raíz del repositorio, en el servidor.

1. [Comprobar los requisitos](#1-comprobar-los-requisitos)
2. [Preparar las carpetas](#2-preparar-las-carpetas)
3. [Configurar OpenClaw y arrancar](#3-configurar-openclaw-y-arrancar)
4. [Abrir el panel](#4-abrir-el-panel)
5. [Configurar los agentes y sus roles](#5-configurar-los-agentes-y-sus-roles)
6. [Registrar las conexiones MCP](#6-registrar-las-conexiones-mcp)
7. [Aplicar los permisos y activar los RAG](#7-aplicar-los-permisos-y-activar-los-rag)

Si OpenClaw ya está configurado y en marcha, empieza en el paso 4. Antes de
aplicar los parches, comprueba que existen los workspaces descritos en el paso 2.
El orden de los parches es **agentes → MCP → permisos**.

## 1. Comprobar los requisitos

Necesitas Linux x86_64, Docker Engine, Compose **2.30 o superior** y una GPU
NVIDIA con su driver y el
[NVIDIA Container Toolkit configurado para Docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-docker).
Ollama e Infinity usan la GPU; Docling trabaja en CPU.

```sh
docker compose version
nvidia-smi
```

El repositorio no instala paquetes ni reconfigura el host.
La primera descarga de imágenes y modelos requiere conexión a Internet.

## 2. Preparar las carpetas

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
límite de RAM de Docling. La [referencia de variables](arquitectura.md#variables-de-entorno)
recoge sus valores por defecto. Después valida el Compose y descarga las imágenes:

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

## 3. Configurar OpenClaw y arrancar

OpenClaw necesita dos autenticaciones distintas:

- **Proveedor del modelo:** su API key o inicio de sesión, según el proveedor.
  El asistente permite configurarlo. `bge-m3` sirve para embeddings; el modelo
  de conversación de OpenClaw se elige por separado.
- **Panel web:** el asistente genera un token del Gateway automáticamente.
  Es normal que no te pida una contraseña para el panel. En el paso siguiente
  obtendrás el enlace de acceso sin buscar ni copiar ese token.

En esta fase el servicio `openclaw` debe estar detenido. Si estás repitiendo
el asistente en una instalación existente, sigue primero las indicaciones para
[cambiar el modelo o sus credenciales](uso.md#cambiar-el-modelo-o-sus-credenciales).

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

## 4. Abrir el panel

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

## 5. Configurar los agentes y sus roles

Tras configurar el proveedor del modelo en el paso 3, aplica
[`config/openclaw-agents.json`](../config/openclaw-agents.json). Conserva el agente
actual con ID `main` como **Personal** y añade los otros dos:

| ID | Nombre y función | Workspace dentro de `data/openclaw/` |
| --- | --- | --- |
| `main` | **Personal**: organización, tareas y planes personales; agente por defecto | `workspace/` |
| `universidad` | **Universidad**: tutoría, apuntes, ejercicios y estudio | `workspace-universidad/` |
| `tecnico-homelab` | **Técnico-Homelab**: sistemas, redes, contenedores y diagnóstico | `workspace-tecnico-homelab/` |

Los roles están en los archivos `SOUL.md` de [`config/agents/`](../config/agents/).
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

## 6. Registrar las conexiones MCP

[`config/openclaw-mcp.json`](../config/openclaw-mcp.json) conecta OpenClaw con los
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

## 7. Aplicar los permisos y activar los RAG

Aplica [`config/openclaw-permissions.json`](../config/openclaw-permissions.json)
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

Con los tres agentes configurados, continúa con
[añadir documentos y comprobar la ingesta](uso.md#añadir-documentos).
