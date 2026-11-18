# Instrucciones para asistentes de código

Este repositorio es un laboratorio de seguridad ofensiva para formación interna. Se ejecuta
únicamente en local o contra activos propios con autorización.

- **Alcance permitido:** `127.0.0.1`, red Docker local, servicios de práctica autoalojados, y este
  código. Nada de internet de terceros.
- **Idiomas:** documentación en español; ramas, commits y código en inglés.
- **Flujo de ramas:** `dev` integra; el trabajo nace en `feat/*` o `fix/*` y se fusiona con `--no-ff`.
  `main` es espejo del upstream.
- **Verificación:** ejecuta `go test ./...`, `npm test` (en `web/`) y `npm run build:static` antes de
  afirmar que algo funciona. Reporta lo que no pudiste probar.

Contexto completo en [`AGENTS.md`](../AGENTS.md). Documentación en [`docs/indice.md`](../docs/indice.md).
