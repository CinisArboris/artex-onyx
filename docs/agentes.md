# Agentes: cómo añadirlos y qué los hace funcionar

Español · [← índice de documentación](indice.md)
Un agente en ScopeWeaver es **un cerebro con instrucciones** conectado a **manos que ejecutan**. Crear el agente es el 20% del trabajo; el otro 80% es darle manos.

---

## 1. Cerebro y músculo

| Capa | Qué es | Dónde se edita | Sin esto… |
| --- | --- | --- | --- |
| **Cerebro** | Prompt del sistema, variables, límites de turnos/tiempo | *Sistema → Agente → Configuración* | El agente no sabe qué rol tiene |
| **Músculo** | Herramientas (tools) nativas y personalizadas | *Sistema → Agente → Herramientas* / *Sistema → Herramientas* | Piensa pero no puede tocar nada |
| **Memoria de procedimiento** | Skills (`SKILL.md`), invocables bajo demanda | *Sistema → Skills* o `skills/` en disco | Repite pasos que ya documentaste |
| **Extensiones externas** | Servidores MCP | *Sistema → MCP* | Sin acceso a sistemas que no son HTTP/CLI |
| **Perfil LLM** | Qué modelo razona | *Sistema → LLM*, o el campo del agente | No hay inferencia en absoluto |

Regla práctica: **si el agente "no hace nada", no es el prompt.** En el 90% de los casos falta un perfil LLM activo o la tool no está atada al agente.

---

## 2. Los agentes que ya existen

Seis son de código (`db/db.go:184`) y dos se siembran como agentes personalizados:

| Key | Nombre | Rol | Para qué |
| --- | --- | --- | --- |
| `goals` | Descomposición de objetivos | `goals` | Parte un objetivo en subobjetivos verificables |
| `planner` | Planificador | `planner` | Añade intenciones de exploración sobre huecos reales |
| `mainagent` | Agente principal | `main` | Tu interfaz: observa y convierte tu intención en pistas |
| `worker` | Trabajador | `worker` | Reclama una intención, la ejecuta y escribe hechos |
| `auto` | Auto | `assistant` | Asistente de plataforma: tareas, activos, skills, MCP |
| `pentest` | Pruebas de penetración | `assistant` | Pentest autónomo de punta a punta por chat |
| `reporter` | Redactor de informes | `assistant` | Se dispara con cada `report_finding` |
| `retester` | Comprobación de hallazgos | `assistant` | Reejecuta la PoC de un hallazgo |

Los agentes de código no se pueden borrar ni renombrar desde la UI, pero **su prompt sí es editable** y tiene botón de reset al valor por defecto.

---

## 3. Crear un agente por la UI

*Sistema → Agente → Crear.* Solo pide tres datos:

| Campo | Regla | Notas |
| --- | --- | --- |
| `key` | `^[a-z][a-z0-9_]*$` | Inmutable. Es la clave de API y de BD. No usa guiones. |
| Nombre | No vacío | Visible en el selector de conversaciones |
| Descripción | Opcional | Aparece en la tarjeta y ayuda al modelo en la selección |

El backend valida la clave contra `server/server_mgmt.go:67`, rechaza duplicados con 409 y siembra un prompt inicial editable (`DefaultAssistantPrompt`) para que el editor no esté vacío.

Valores por defecto que hereda todo agente nuevo (`db/schema.sql:778`):

| Columna | Default | Significado |
| --- | --- | --- |
| `role` | `assistant` | Familia de comportamiento del motor |
| `builtin` | `false` | Se puede borrar |
| `enabled` | `true` | Activo desde el minuto uno |
| `max_turns` | `0` | Sin límite de turnos |
| `run_seconds` | `1200` | 20 minutos por ejecución |
| `web_search` | `false` | Sin búsqueda web |
| `interactive_shell` | `false` | Sin shell persistente |
| `trigger_run_mode` | `serial` | Disparos en serie |
| `trigger_merge_mode` | `all` | Mezcla todas las salidas |
| `trigger_max_parallel` | `5` | Tope de disparos simultáneos |

---

## 4. Pestañas del editor, una por una

**Configuración** — aquí vive el cerebro:

- **Prompt**: el texto del sistema. Admite variables `{{.Nombre}}`; el botón de vista previa las renderiza antes de guardar. Hay historial de versiones y *reset* al default.
- **Variables**: catálogo de variables con descripción y ejemplo. Solo se aceptan nombres declarados; una variable inventada no resuelve.
- **Perfil LLM**: vacío = hereda. Si eliges uno, este agente ignora el global.
- **Máx. turnos** / **Segundos por ejecución**: presupuesto duro. `0` en turnos = sin límite.
- **Búsqueda web**: interruptor global en *Sistema → Ajustes*; el del agente solo tiene efecto si el global está activo.
- **Shell interactivo**: permite sesiones Bash persistentes entre turnos. Desactivado por defecto a propósito.
- **Prompt de cierre (wrap-up)**: se añade cuando se agota el presupuesto, para que el agente guarde conclusiones en vez de morir a medias.

**MCP / Skills / Herramientas** — aquí vive el músculo. Son visibilidades, no copias: marcar una casilla expone el recurso a ese agente.

> **Trampa habitual:** un agente nuevo nace **sin ninguna tool de dominio**. Las tools de pentest solo se atan por defecto a las claves builtin (`pentest`, `auto`). Si tu agente nuevo no registra activos ni hallazgos, no es que falle: no tiene la tool.

Tools que **todo** agente tiene siempre (vienen del SDK, no de la BD): `Read`, `Write`, `Edit`, `MultiEdit`, `LS`, `Glob`, `Grep`, `Bash`.

Tools de dominio que merece la pena atar a un agente de pentest (`agent/toolcatalog.go:38`):

| Tool | Para qué |
| --- | --- |
| `insert_assets` | Registrar endpoints, fingerprints, servicios |
| `list_assets` | Consultar antes de duplicar |
| `add_finding` | Reportar vulnerabilidad confirmada con PoC |
| `list_findings` | Evitar duplicados |
| `list_companies` | Consultar el alcance declarado |
| `add_company_scope` | Ampliar alcance (sujeto a reglas de activos) |

`goal_met` existe pero **no** se ata por defecto: puede cerrar una tarea entera saltándose las pruebas. Úsalo solo con conocimiento de causa.

---

## 5. Crear un agente por API

Requiere el JWT de administrador (misma sesión que la UI):

```bash
# 1) Crear
curl -X POST http://127.0.0.1:8787/api/agents \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"key":"lab_recon","name":"Recon de laboratorio","description":"Reconocimiento sobre alcance autorizado"}'

# 2) Configuración
curl -X PUT http://127.0.0.1:8787/api/agents/lab_recon/config \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"max_turns":60,"run_seconds":900,"interactive_shell":true,"llm_profile_id":null}'

# 3) Prompt
curl -X PUT http://127.0.0.1:8787/api/agents/lab_recon/prompt \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"template":"Eres un analista de reconocimiento..."}'

# 4) Visibilidad de MCP y skills
curl -X PUT http://127.0.0.1:8787/api/agents/lab_recon/visibility \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"mcp":[],"skill":["api-recon"]}'
```

Las tools se atan por el catálogo, no por el agente: `PUT /api/tools/{key}` con el campo `agents`. Ejemplo, dar `insert_assets` al agente nuevo:

```bash
curl -X PUT http://127.0.0.1:8787/api/tools/insert_assets \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"description":"","schema":{},"agents":["pentest","auto","lab_recon"],"enabled":true}'
```

`POST /api/tools/{key}/reset` devuelve una tool a su descripción, esquema y bindings de código.

---

## 6. Skills: memoria de procedimiento

Una skill es una carpeta con `SKILL.md` en formato [agentskills.io](https://agentskills.io): frontmatter YAML y cuerpo Markdown.

```
skills/
  api-recon/
    SKILL.md          # frontmatter + método
    scripts/          # plantillas ejecutables
    reference.md      # recetas largas, fuera del contexto del agente
```

```markdown
---
name: mi-metodo
description: Úsala para auditar cabeceras de seguridad en alcance autorizado.
---

# Mi método

1. Paso concreto.
2. Criterio de parada.
```

Reglas que importan:

- `description` es lo que el modelo lee para decidir si invoca la skill. Si es vaga, nunca se usará.
- El cuerpo se carga **solo al invocar**, así que no infla el contexto en cada turno. Por eso las recetas largas van en un archivo aparte.
- La raíz se resuelve por `ARTEX_SKILL_DIR`, luego `skill_dir` de `config.json`, luego `skills/` junto al binario. En Docker Compose está montada desde `./skills` del host: editas en tu disco y reinicias.
- Los identificadores y comandos quedan en su idioma original; no traduzcas contenido de usuarios ni rutas.

La carpeta `docs/habilidades/` contiene **traducciones de referencia** de las skills en inglés. Las entradas ejecutables son siempre las de `skills/` en la raíz del repo. Esa es la razón de que `docs/` no tenga una carpeta `skills/` en inglés: el original vive en la raíz porque el binario lo lee desde ahí.

---

## 7. Qué modelo usa cada agente

Orden de resolución, del más específico al más general:

1. **Perfil del agente** (`llm_profile_id` en el agente).
2. **Cadena de la tarea**, si la tarea define perfiles.
3. **Perfil global activo** (*LLM → Activar*).

La conversación también puede fijar su propio perfil al crearse. Si el global está vacío y ningún nivel superior define uno, el agente no arranca: `llm_configured` es `false`.

---

## 8. Verificar que quedó bien

```sql
-- Agentes y su estado
SELECT key, name, builtin, enabled, max_turns, run_seconds, llm_profile_id
FROM agents ORDER BY id;

-- Tools atadas a tu agente
SELECT key, agents FROM tools WHERE 'lab_recon' = ANY(agents);
```

En la UI: el agente aparece en el selector de *Chat → Crear conversación*. Si arranca y no produce acciones, revisa en este orden: perfil LLM activo → tools atadas → MCP visible → prompt con criterio de parada.

---

## 9. Checklist para una PoC que demuestre algo

- [ ] Perfil LLM creado **y activado** (no basta con guardarlo).
- [ ] Alcance declarado con el target de laboratorio.
- [ ] Agente con prompt que define objetivo y criterio de parada.
- [ ] Al menos `insert_assets` y `add_finding` atadas.
- [ ] Una skill propia que documente tu método.
- [ ] Una conversación real con evidencia: activo registrado + hallazgo con PoC reproducible.
- [ ] Reglas de intercepción revisadas: sin ellas no hay barrera de red real.

Las restricciones de uso siguen vigentes: solo entornos aislados localmente o con autorización explícita. Ver [licencia y exención de responsabilidad](../README.es.md#licencia-y-exención-de-responsabilidad).
