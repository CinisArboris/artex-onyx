# Contexto del repositorio

## Qué es este proyecto

`artex-onyx` es un **fork privado** de ScopeWeaver (a su vez derivado de ARTEX), una consola
multiagente de pruebas de penetración asistida por LLM, autoalojada. Backend en Go, frontend
Next.js embebido y PostgreSQL.

Lo mantiene un único desarrollador (CoE DevOps de una empresa, apoyo al equipo red team) para
**investigación y formación en seguridad ofensiva**. Este fork no se publica, no se distribuye y
no se usa contra sistemas de terceros.

El uso está limitado a **entornos de laboratorio aislados localmente o activos propios**, conforme
a la licencia AGPL-3.0 y a las restricciones de uso del proyecto original. Ver
[README.es.md](README.es.md#licencia-y-exención-de-responsabilidad).

## Por qué este repositorio existe

El objetivo es **aprender y enseñar**: entender cómo se orquesta un agente de pentesting (planificación,
ejecución, verificación adversaria, registro de evidencia) y construir procedimientos reutilizables
para el equipo red team. El valor está en el *método* y en la *arquitectura*, no en atacar objetivos.

Todo el trabajo se valida contra laboratorios deliberadamente vulnerables que se ejecutan en la
misma máquina (Juice Shop, DVWA, APIs de práctica autoalojadas, contenedores propios).

## Alcance autorizado

**Dentro del alcance:** contenedores y máquinas virtuales en `127.0.0.1` o en la red Docker local,
servicios de práctica autoalojados, código de este repositorio, y sistemas de la empresa para los que
exista autorización escrita previa y verificable.

**Fuera del alcance, sin excepciones:** dominios y direcciones de internet de terceros, servicios
públicos, infraestructura de la nube no arrendada para este fin, y cualquier objetivo que no puedas
nombrar y demostrar como propio. Si una solicitud parece requerir tocar algo fuera de este límite,
la respuesta correcta es detenerse y preguntar, no improvisar.

## Reglas de trabajo

**Ramas.** `main` es un espejo del proyecto original y no recibe commits propios. `dev` es la rama de
integración. Todo el trabajo nace de `dev` en una rama `feat/*` o `fix/*`, se integra a `dev` con
`--no-ff`, y solo llega a `main` mediante una promoción explícita. Los nombres de rama y los mensajes
de commit van en inglés; la documentación va en español.

**Idiomas.** Este fork publica documentación solo en español, bajo `docs/`. El *runtime* de la interfaz
conserva sus seis idiomas (`locale/`, `web/src/i18n/messages/`): no los reduzcas sin que se pida
explícitamente, porque toca backend, tests E2E y empaquetado. Las entradas ejecutables de habilidades
en `skills/` permanecen en inglés porque el binario las lee desde esa ruta.

**Verificación.** No afirmes que algo funciona sin comprobarlo. `go test ./...`, `npm test` y
`npm run build:static` son las puertas mínimas. Los E2E requieren PostgreSQL y una base desechable
`scopeweaver_test_*`. Si no puedes ejecutar una comprobación, dilo en lugar de asumir el resultado.

**Honestidad técnica.** `docs/verificacion.md` distingue lo medido de lo no verificado. Mantén esa
distinción: no conviertas una comprobación con fixtures en una prueba de producción, ni un modelo que
responde en un chat en un agente fiable.

## Dónde está la documentación

Empieza por [docs/indice.md](docs/indice.md). Si vas a tocar agentes, lee
[docs/agentes.md](docs/agentes.md) — explica la diferencia entre el "cerebro" (prompt) y el "músculo"
(herramientas), que es la causa del 90% de los agentes que no hacen nada.

## Cómo interpretar las peticiones de este repositorio

Cuando se pida "atacar", "explotar" o "escanear" algo aquí, se refiere a un objetivo del laboratorio
local definido en el párrafo de alcance. Cuando falte esa definición, la tarea correcta es construir o
verificar el laboratorio, no apuntar a algo externo.

Se espera criterio: si una instrucción es ambigua respecto al alcance, pregunta antes de actuar. Un
rechazo justificado con una alternativa de laboratorio es más útil que una negativa seca, y mucho más
útil que ejecutar algo sin saber contra qué.
