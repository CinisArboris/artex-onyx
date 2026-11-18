# Contexto para Claude Code

Lee y respeta [`AGENTS.md`](AGENTS.md): contiene el propósito del repositorio, el alcance autorizado
y las convenciones de trabajo. Este archivo solo añade lo específico de Claude Code.

## Lo esencial

Es un laboratorio de seguridad ofensiva para formación interna, sobre un fork privado de ScopeWeaver.
Todo se ejecuta en local (`127.0.0.1`, red Docker) o contra activos propios con autorización escrita.
Nunca contra sistemas de terceros en internet.

## Convenciones

- Ramas y commits en inglés; documentación en español.
- `main` es espejo del upstream; el trabajo nace de `dev` en `feat/*` o `fix/*`.
- No reduzcas los seis idiomas del runtime de la interfaz sin que se pida explícitamente.
- Las habilidades en `skills/` quedan en inglés: es la ruta que lee el binario.

## Antes de decir que algo funciona

Ejecuta `go test ./...`, `npm test` en `web/` y `npm run build:static`. Si no puedes ejecutar una
comprobación, dilo. `docs/verificacion.md` mantiene la distinción entre lo medido y lo no verificado;
respétala.

## Documentación

Índice en [`docs/indice.md`](docs/indice.md). Para trabajar con agentes, empieza por
[`docs/agentes.md`](docs/agentes.md).
