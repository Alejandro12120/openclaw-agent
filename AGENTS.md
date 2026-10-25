# Trabajo en este repositorio

- Mantener PR pequeñas. Esta primera fase contiene la infraestructura Docker.
- Usar Docker Compose 2.30 o superior; no instalar dependencias en el host.
- Validar los cambios con `docker compose config --quiet` y `git diff --check`.
- Verificar el arranque con datos temporales, sin modificar `data/`, `rag/`
  ni los archivos del usuario en `documents/`.
- Los datos de `data/`, `rag/`, `documents/` y `.env` no se versionan.
- Un único OpenClaw gestionará tres agentes. Sus permisos RAG están pendientes;
  no afirmar que existe aislamiento entre agentes por tener bases separadas.
- Un único haiku-ingester es el escritor de las tres bases; no arrancar otro
  escritor contra ellas. Los tres MCP comparten Ollama y el servidor de reranking.
- `documents/` se monta en solo lectura; el ingester guarda bases y cachés en
  `rag/`. Docling es un servicio separado con imagen publicada; no se mantienen
  Dockerfiles propios. Los PDF se procesan en bloques de 10 páginas.
- El Compose usa NVIDIA para Ollama e Infinity y CPU para Docling. Requiere
  NVIDIA Container Toolkit configurado; no instalarlo ni reconfigurar Docker
  automáticamente en el host. Usar `.agent/tmp/` para pruebas con modelos pesados.
