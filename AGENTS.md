# Trabajo en este repositorio

- Mantener PR pequeñas y separar la conexión MCP de los permisos por agente.
- Usar Docker Compose 2.30 o superior; no instalar dependencias en el host.
- Validar los cambios con `docker compose config --quiet` y `git diff --check`.
- Verificar el arranque con datos temporales, sin modificar `data/`, `rag/`
  ni los archivos del usuario en `documents/`.
- Los datos de `data/`, `rag/`, `documents/` y `.env` no se versionan.
- Un único OpenClaw gestionará tres agentes. Sus permisos RAG están pendientes;
  no afirmar que existe aislamiento entre agentes por tener bases separadas.
- `config/openclaw-mcp.json` registra los tres MCP desactivados. Mantener el
  filtro de herramientas de lectura; no habilitar `execute_code`. La activación
  debe acompañarse de permisos por agente y pruebas de accesos denegados.
- `main` conserva el agente Personal existente; los otros IDs son `universidad`
  y `tecnico-homelab`. Sus roles se versionan en `config/agents/*/SOUL.md` y se
  montan en solo lectura. Aplicar `config/openclaw-agents.json` con `config patch`,
  primero con `--dry-run`, para conservar la configuración y el estado existentes.
  Con el Gateway arrancado, usar `docker compose exec` en su contenedor: un
  `compose run` separado no puede verificar el bloqueo de estado entre namespaces.
- Un único haiku-ingester es el escritor de las tres bases; no arrancar otro
  escritor contra ellas. Los tres MCP comparten Ollama y el servidor de reranking.
- `documents/` se monta en solo lectura; el ingester guarda bases y cachés en
  `rag/`. Docling es un servicio separado con imagen publicada; no se mantienen
  Dockerfiles propios. Los PDF se procesan en bloques de 10 páginas.
- El Compose usa NVIDIA para Ollama e Infinity y CPU para Docling. Requiere
  NVIDIA Container Toolkit configurado; no instalarlo ni reconfigurar Docker
  automáticamente en el host. Usar `.agent/tmp/` para pruebas con modelos pesados.
