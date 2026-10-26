# Uso y mantenimiento

[← Inicio](../README.md) · [Instalación](instalacion.md) · [Arquitectura](arquitectura.md)

Con el despliegue en marcha, puedes consultar tus documentos personales,
de la universidad y del homelab con el agente correspondiente. Esta guía explica
cómo añadir documentos, hacer consultas y mantener los servicios.
Ejecuta los comandos desde la raíz del repositorio.

- [Añadir documentos](#añadir-documentos)
- [Consultar documentos](#consultar-documentos)
- [Trabajar con PDF grandes](#pdf-grandes)
- [Consultar y controlar los servicios](#operaciones-habituales)
- [Cambiar el modelo o sus credenciales](#cambiar-el-modelo-o-sus-credenciales)
- [Actualizar los roles](#actualizar-los-roles)
- [Hacer copias de seguridad](#copias-de-seguridad)
- [Resolver problemas](#resolución-de-problemas)

## Añadir documentos

Coloca cada archivo en la carpeta de su ámbito:

| Carpeta de originales | Base RAG | Agente que la consulta |
| --- | --- | --- |
| `documents/personal/` | `personal` | Personal |
| `documents/universidad/` | `universidad` | Universidad |
| `documents/tecnico/` | `tecnico-homelab` | Técnico-Homelab |

La carpeta de origen determina en qué base se indexa el archivo. Cada agente
solo tiene herramientas para consultar la base de su ámbito.

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

## Consultar documentos

Cuando la ingesta termine, abre **Agents** en el panel y usa **Open chat** para
abrir un chat nuevo con el agente del ámbito que quieras consultar.
Puedes pedirle, por ejemplo: «Busca en mis documentos qué se explica sobre
[tema] y cita las fuentes». También puedes pedir resúmenes o comparar contenido
de documentos de esa misma base. Para consultar otro ámbito, abre el chat del
agente correspondiente; ninguno puede buscar en las bases de los otros dos.

Las respuestas deben apoyarse en los documentos consultados e indicar si falta
información para contestar. Comprueba que haya una llamada a la herramienta
`rag-…__search_documents` antes de la respuesta y que la referencia corresponda
al contenido consultado.

## PDF grandes

En [`config/ingester.yaml`](../config/ingester.yaml), `processing.split_pages: 10`
convierte los PDF por bloques de diez páginas y une el resultado conservando
la numeración. Por ejemplo, un PDF de 21 páginas se procesa en bloques de 10, 10 y 1.

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

`docker compose down` conserva las carpetas de datos del host. Para recuperar
el acceso al panel, genera otro enlace como se explica en
[abrir el panel](instalacion.md#4-abrir-el-panel).

## Cambiar el modelo o sus credenciales

El modelo de conversación se configura en OpenClaw, por separado del embedder
y del reranker. Los agentes nuevos heredan su modelo por defecto, salvo que
configures uno específico para cada agente.

Para repetir el asistente interactivo con `docker compose run`, detén primero
el Gateway y evita abrir su estado desde dos contenedores a la vez:

```sh
docker compose stop openclaw
```

Ejecuta el [asistente de configuración](instalacion.md#3-configurar-openclaw-y-arrancar),
conserva los valores que no necesites cambiar y vuelve a arrancar OpenClaw:

```sh
docker compose up -d --no-deps --wait openclaw
```

Si cambias de modelo, comprueba de nuevo que consulta el RAG y cita sus fuentes.
El contexto disponible debe dar cabida a las instrucciones, el historial,
las herramientas, los fragmentos recuperados y la respuesta.

## Actualizar los roles

Edita el `SOUL.md` correspondiente en [`config/agents/`](../config/agents/).
Los roles se montan en solo lectura dentro de OpenClaw; para cargar el cambio:

```sh
docker compose up -d --no-deps --force-recreate --wait openclaw
```

Abre un chat nuevo con el agente. Si solo has cambiado los roles, no necesitas
reaplicar los parches de agentes, MCP o permisos.

## Copias de seguridad

Conserva una copia de `.env`, `data/`, `documents/` y `rag/`. Ahí están los
ajustes locales, las credenciales, las sesiones, las notas, los documentos,
las bases y las cachés de modelos. Consulta el
[mapa de almacenamiento](arquitectura.md#dónde-se-guardan-los-datos) para identificar cada ruta.

Para obtener una copia coherente de las bases y el estado, detén los servicios
antes de copiar esas carpetas con tu herramienta de backup habitual:

```sh
docker compose stop
```

Una vez terminada la copia, vuelve a arrancarlos:

```sh
docker compose up -d --wait --wait-timeout 1200
```

Estas carpetas están excluidas de Git; publicar el repositorio no sustituye
la copia de seguridad.

## Resolución de problemas

### El panel no abre o el enlace ha caducado

Genera un enlace nuevo y abre el campo `browserUrl`:

```sh
docker compose exec openclaw node dist/index.js dashboard --json
```

El enlace es de un solo uso y caduca a los diez minutos. Si accedes desde otro
equipo o cambiaste el puerto, sigue las indicaciones de
[acceso al panel y túnel SSH](instalacion.md#4-abrir-el-panel).

### El agente no encuentra sus documentos

1. Comprueba que el archivo esté en la carpeta del ámbito correcto y revisa
   los logs del ingester con los comandos de [añadir documentos](#añadir-documentos).
2. Revisa las conexiones y su diagnóstico:

   ```sh
   docker compose exec openclaw node dist/index.js mcp status --verbose
   docker compose exec openclaw node dist/index.js mcp probe --json
   ```

3. Si las conexiones están desactivadas, revisa el orden de los
   [parches de instalación](instalacion.md#6-registrar-las-conexiones-mcp).
   Reaplicar el registro MCP las desactiva hasta aplicar de nuevo los permisos.
4. Abre un chat nuevo del agente correcto y verifica que su lista de herramientas
   incluya el prefijo de su RAG. El diagnóstico del Gateway y el acceso del agente
   son comprobaciones distintas.

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
solo lectura. Para evitarlo en instalaciones nuevas,
[crea los directorios](instalacion.md#2-preparar-las-carpetas) con tu usuario
antes de arrancar Compose.
