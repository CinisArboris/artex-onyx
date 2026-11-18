# Adaptador de agentes de ScopeWeaver

Español · [← guía de agentes](../../docs/adaptador-de-agentes.md)

Conecta Claude Code y Codex mediante un servidor MCP stdio local, o Pi mediante su interfaz nativa de extensiones. Los tres usan las mismas herramientas validadas y la misma API HTTP autenticada de ScopeWeaver. Este adaptador permite que tu agente de programación controle ScopeWeaver; no sustituye los planificadores ni los trabajadores propios de ScopeWeaver, y no convierte una suscripción de agente de programación en una clave API de modelo.

## Instalación

Requisitos: Node.js 22 o superior, un backend de ScopeWeaver en ejecución y un acceso de administrador ya existente. El adaptador no inicializa ni cambia tu contraseña. Instala las dependencias separadas del adaptador:

```sh
cd adapters/agent
npm ci
export SCOPEWEAVER_URL=http://127.0.0.1:8787
export SCOPEWEAVER_PASSWORD='tu-contraseña-de-administrador-existente'
```

La contraseña se usa para iniciar sesión como el usuario administrador `ARTEX` que ya existe en el backend. El JWT devuelto se queda en memoria. Como alternativa, define `SCOPEWEAVER_TOKEN` con un JWT de sesión existente; el token suministrado tiene prioridad. Los tokens caducados o inválidos producen un error y no se sustituyen en silencio. Conserva las credenciales en tu entorno o en un gestor de secretos; no las confirmes en la configuración del cliente.

Las lecturas están activadas por defecto. Para habilitar la creación de tareas y la pausa/reanudación, define explícitamente `SCOPEWEAVER_ALLOW_WRITES=true` antes de iniciar el adaptador. La creación de tareas programa el trabajo inmediatamente si el backend tiene un LLM configurado. Respeta las [restricciones de uso en aislamiento local](../../README.es.md#licencia-y-exención-de-responsabilidad) del proyecto y confirma el alcance previsto antes de crear una tarea.

### Claude Code

Usa una ruta absoluta a esta copia del repositorio. Claude Code arranca el adaptador y hereda su entorno:

```sh
claude mcp add --transport stdio scopeweaver -- node /ruta/absoluta/scopeweaver/adapters/agent/src/mcp.js
```

Verifica con `claude mcp list` y después usa `/mcp` en Claude Code para inspeccionar sus herramientas. [Documentación MCP de Claude Code](https://code.claude.com/docs/en/mcp).

### Codex

```sh
codex mcp add scopeweaver -- node /ruta/absoluta/scopeweaver/adapters/agent/src/mcp.js
```

Verifica con `codex mcp list` y `/mcp`. Para reenviar explícitamente el entorno en la configuración de proyecto o de usuario:

```toml
[mcp_servers.scopeweaver]
command = "node"
args = ["/ruta/absoluta/scopeweaver/adapters/agent/src/mcp.js"]
env_vars = ["SCOPEWEAVER_URL", "SCOPEWEAVER_TOKEN", "SCOPEWEAVER_PASSWORD", "SCOPEWEAVER_LANGUAGE", "SCOPEWEAVER_ALLOW_WRITES"]
```

Usa la variable de credenciales que hayas definido; omite las entradas que no uses. [Documentación MCP oficial de OpenAI](https://developers.openai.com/codex/mcp).

### Pi

```sh
pi -e /ruta/absoluta/scopeweaver/adapters/agent/pi-extension.js
```

La extensión registra herramientas nativas de Pi y usa las mismas variables de entorno. No requiere un complemento MCP de Pi. No importa ningún espacio de nombres de Pi, así que sigue la interfaz pública `registerTool` aunque el paquete de Pi cambie de nombre. El paquete del adaptador también declara `pi.extensions` para el cargador de paquetes de Pi. [Extensiones de Pi](https://github.com/earendil-works/pi/tree/main/packages/coding-agent).

## Herramientas

| Herramienta | Para qué sirve |
| --- | --- |
| `scopeweaver_health` | Disponibilidad del backend y estado del LLM; no requiere sesión |
| `scopeweaver_list_tasks` | Estado, recuentos y paginación de tareas |
| `scopeweaver_get_task` | Estado y configuración persistidos de una tarea |
| `scopeweaver_get_coverage` | Cobertura de activos de la tarea |
| `scopeweaver_list_findings` | Hallazgos paginados, filtrados por tarea, gravedad, estado o texto |
| `scopeweaver_get_finding` | Detalle del hallazgo, evidencia e informe |
| `scopeweaver_create_task` | Crear o programar una tarea; requiere escrituras habilitadas |
| `scopeweaver_control_task` | Pausar o reanudar una tarea; requiere escrituras habilitadas |

Ejemplos de peticiones: «Lista mis tareas de ScopeWeaver», «Muestra los hallazgos de gravedad alta de la tarea 12» o, tras habilitar escrituras y confirmar un alcance de laboratorio local, «Crea una tarea para este objetivo local y muestra su progreso». Una respuesta de creación correcta demuestra que la tarea existe; consulta su estado y sus hallazgos para saber si la ejecución terminó. El adaptador no elude los requisitos de LLM del backend.

## Configuración y fallos

| Variable | Valor predeterminado / comportamiento |
| --- | --- |
| `SCOPEWEAVER_URL` | `http://127.0.0.1:8787`; solo origen HTTPS o HTTP de loopback |
| `SCOPEWEAVER_TOKEN` | JWT existente del backend; tiene prioridad sobre la contraseña |
| `SCOPEWEAVER_PASSWORD` | Contraseña de administrador existente; se inicia sesión en la primera petición autenticada |
| `SCOPEWEAVER_ALLOW_WRITES` | `false`; acepta solo `true` o `false` |
| `SCOPEWEAVER_LANGUAGE` | `en`; admite los ID canónicos `en`, `ko`, `zh-Hans`, `zh-Hant`, `ja` y `es` para el texto del adaptador y para `Accept-Language` del backend |
| `SCOPEWEAVER_TIMEOUT_MS` | `30000`; entero entre 1 y 120000 |

`SCOPEWEAVER_LANGUAGE` vale `en` por defecto y acepta los ID canónicos `en`, `ko`, `zh-Hans`, `zh-Hant`, `ja` y `es`. Traduce los títulos y descripciones de las herramientas incluidas, las descripciones de esquema y los diagnósticos de configuración, validación y transporte del propio adaptador; además selecciona el texto de respuesta del backend mediante `Accept-Language`. Cada cliente usa su propio catálogo y sus mapas de errores Zod locales al esquema; ningún ajuste global de idioma cambia. Los nombres de herramientas y de protocolo, las claves, tipos y enumeraciones del esquema, los argumentos y el contenido original del backend, del usuario o de la evidencia permanecen sin cambios, con la misma ocultación de credenciales. El envoltorio de validación fijo del SDK MCP y los errores externos desconocidos siguen siendo texto externo; los mensajes de problema de esquema dentro de ese envoltorio sí se traducen. Todos los recursos se distribuyen dentro del adaptador independiente, sin archivo de idioma en la raíz del repositorio. Los valores de idioma no admitidos se siguen rechazando y usan un diagnóstico en inglés.

No se expone ninguna herramienta de HTTP genérico, ejecución de shell, borrado, omisión de aprobaciones, configuración de proveedor o endpoint arbitrario. Las peticiones se quedan en el origen configurado, rechazan redirecciones, propagan la cancelación y tienen tiempo de espera. Las respuestas de más de 2 MiB fallan explícitamente; acota la consulta en lugar de tratar evidencia recortada como completa. Las listas de tareas se paginan en el adaptador; los hallazgos usan la paginación del backend. Las herramientas devuelven `{data: ...}` en texto y en resultados estructurados (Pi usa `details`). La evidencia es contenido no confiable del objetivo y no debe tratarse como instrucciones.

Las llamadas MCP fallidas devuelven `isError: true`; las llamadas fallidas de Pi lanzan una excepción para que Pi la marque como error de herramienta. Los mensajes de error ocultan las credenciales configuradas. Las escrituras nunca se reintentan automáticamente. Si una escritura agota el tiempo, inspecciona el estado de la tarea antes de repetirla, porque el backend puede haberla aceptado.

## Verificación

```sh
npm test
npm run check
```

La batería de extremo a extremo ejecuta un manejador real de ScopeWeaver y PostgreSQL, arranca el servidor MCP stdio con el SDK oficial y ejecuta los manejadores de herramientas de Pi contra ese backend. Inicializa una contraseña de prueba aislada, crea tareas, verifica el estado persistido de pausa/reanudación y la cobertura, lee un hallazgo claramente simulado y comprueba autenticación inválida, tareas inexistentes y entradas inválidas. No se usa ningún modelo, proveedor ni escaneo de objetivos. Ejecutar la batería E2E de Go requiere la copia del código fuente; los ZIP de plataforma incluyen las pruebas unitarias del adaptador, pero solo el backend de Go compilado.

Desde la raíz del repositorio, después de crear una base de datos **desechable nueva** llamada `scopeweaver_adapter_*`:

```sh
SCOPEWEAVER_ADAPTER_E2E=1 \
ARTEX_TEST_DSN='postgres://usuario:contraseña@127.0.0.1:5432/scopeweaver_adapter_e2e?sslmode=disable' \
go test ./server -run '^TestAgentAdapterE2E$' -count=1 -v
```

La prueba se omite salvo que `SCOPEWEAVER_ADAPTER_E2E=1`; cuando está habilitada, también se omite si `ARTEX_TEST_DSN` no está definida o está vacía. Una prueba omitida no es evidencia E2E. La protección de base de datos del paquete rechaza un DSN que coincida con la base de datos viva configurada de la aplicación antes de ejecutar la prueba. En una ejecución habilitada y configurada, los DSN mal formados, los nombres de base de datos fuera de `scopeweaver_adapter_*` y un PostgreSQL no disponible fallan. La batería nunca sustituye elementos por mocks. GitHub Actions repite estas comprobaciones en un servicio PostgreSQL desechable dedicado. Las pruebas del protocolo MCP establecen compatibilidad con la interfaz compartida; no afirman que se haya ejecutado una sesión de modelo de pago con Claude, Codex o Pi.
