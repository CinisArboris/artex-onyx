# Inferencia local con Ollama

Español · [← índice](../proveedores-llm.md)

Ollama es la vía sin claves y sin salida a la nube. La opción más segura para un laboratorio, y la única que garantiza el tamaño de contexto real.

**Por qué el formato nativo y no `/v1`.** El endpoint compatible con OpenAI de Ollama no permite fijar el tamaño de contexto ([compatibilidad OpenAI](https://docs.ollama.com/api/openai-compatibility)), así que un perfil apuntado a `/v1` no puede garantizar `num_ctx`. El formato nativo usa `/api/chat` y lo envía en cada petición. Una URL base que termine en `/v1`, `/api` o `/api/chat` se normaliza a la raíz del servidor.

**Por qué no confiar en el default de Ollama.** Su documentación no coincide: la [FAQ](https://docs.ollama.com/faq) dice 4.096 tokens y la [página de longitud de contexto](https://docs.ollama.com/context-length) describe un valor según VRAM (unos 4K por debajo de 24 GiB, 32K entre 24 y 48 GiB, 256K por encima). ScopeWeaver fija el valor explícitamente.

---

## Plantillas

**LLM → Nuevo → Plantilla de configuración → Inferencia local (Ollama)**

| Campo | Mismo equipo | Host desde Docker | Servicio Compose incluido |
| --- | --- | --- | --- |
| Formato | Ollama (nativo `/api/chat`) | Igual | Igual |
| URL base | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` | `http://ollama:11434` |
| Modelo | Modelo instalado en ese servidor | Igual | Igual |
| Clave API | vacía | vacía | vacía |
| Ventana de contexto (K) | `32` (`num_ctx` 32.768) | `32` | `32` |
| Peticiones concurrentes máx. | `1` | `1` | `1` |
| Keep-alive | `30m` | `30m` | `30m` |
| Límite de salida | `4096` | `4096` | `4096` |
| Pensamiento / esfuerzo | omitido | omitido | omitido |

Elegir una plantilla Ollama lee `/api/tags` y los metadatos del endpoint elegido, y selecciona un modelo instalado dando preferencia a los que declaran `tools`. El modelo queda vacío mientras se descubre, y sigue vacío con error si el descubrimiento falla o la lista está vacía. **Nunca descarga ni ejecuta un modelo, ni guarda, ni activa un perfil.**

---

## Enrutamiento host y contenedor

| ScopeWeaver corre en | Ollama corre en | URL base |
| --- | --- | --- |
| El host | El mismo host | `http://127.0.0.1:11434` |
| Docker Compose | El host | `http://host.docker.internal:11434` |
| Docker Compose | Servicio `ollama` incluido | `http://ollama:11434` |

Dentro de un contenedor, `127.0.0.1` es el contenedor mismo. Compose mapea `host.docker.internal` a la pasarela del host.

Ollama escucha en `127.0.0.1:11434` por defecto. **En Linux**, el Ollama del host debe escuchar en una dirección que el contenedor alcance (`OLLAMA_HOST=0.0.0.0:11434`), lo que lo expone a la red: restringe el puerto 11434 con un cortafuegos.

**Servicio Compose opcional:** `docker compose --profile local-llm up -d` arranca `ollama` con `OLLAMA_NO_CLOUD=1`, `OLLAMA_KEEP_ALIVE=30m`, `OLLAMA_NUM_PARALLEL=1`, `OLLAMA_CONTEXT_LENGTH=32768` y el volumen `ollama_data`.

```bash
docker compose exec ollama ollama pull qwen2.5-coder:14b
```

Sin el perfil, Compose funciona como antes. El servicio no pide GPU. **En macOS los contenedores Docker no pueden usar la GPU de Apple: este servicio corre solo en CPU.** Para velocidad usable, ejecuta Ollama nativo en el Mac y usa la plantilla de host.

---

## Claves API

- Las plantillas Ollama y cambiar al formato Ollama **borran una clave sin guardar** antes del descubrimiento, para que una credencial de nube nunca se envíe a un endpoint local.
- Conservan nombre, proxy, cabecera de sesión y otras preferencias ya escritas.
- Deja la clave vacía para Ollama local. Introduce un token solo si un proxy inverso delante de Ollama lo exige (se envía como `Authorization: Bearer`).
- Los perfiles sin clave se pueden probar, guardar, activar y usar en failover; la tarjeta muestra **No se necesita clave**.
- Al editar un perfil Ollama nativo guardado, un campo vacío conserva el token guardado. Al pasar un perfil de nube a Ollama con el campo vacío, la clave de nube anterior se elimina.

**Otros formatos sin clave:** Anthropic y OpenAI exigen clave **salvo** que la URL base sea explícitamente local (su host resuelve solo a loopback o direcciones privadas). Un servidor compatible local como vLLM o LM Studio funciona con clave vacía; ScopeWeaver envía un marcador interno no secreto para que el SDK no herede `OPENAI_API_KEY` o `ANTHROPIC_API_KEY` del entorno. Un endpoint no local sin clave se rechaza.

---

## Controles de contexto, salida y memoria

| Ajuste | Comportamiento en Ollama | En la nube |
| --- | --- | --- |
| Ventana de contexto (K) | K binaria enviada como `options.num_ctx` (32 = 32.768). `0` usa 32; valores bajo 16 se elevan a 16. También fija el presupuesto de compactación. | Solo presupuesto local de compactación; `0` = 200K. No se envía. |
| Límite de salida | Enviado como `options.num_predict`; `0` usa 4.096 | `0` envía 32768 en turnos de agente (llamadas directas a Anthropic: 8192) |
| Keep-alive | Enviado como `keep_alive`; vacío usa `30m` (el default de Ollama es 5 minutos) | No se envía |
| Concurrencia máx. | `0` usa 1. Hay una única puerta de admisión por servidor Ollama, incluso entre perfiles; aplica el límite más bajo. | `0` hereda **Sistema → Máx. peticiones LLM concurrentes** |

Solicitudes en cola del juez de aprobación, del agente principal y de preguntas paralelas toman el siguiente hueco libre antes del trabajo de fondo. Cada grupo es FIFO; la inferencia activa nunca se interrumpe.

Sube la concurrencia solo si el servidor tiene `OLLAMA_NUM_PARALLEL` por encima de 1. El uso de memoria de Ollama crece con `OLLAMA_NUM_PARALLEL × longitud de contexto`.

---

## Protecciones durante la ejecución

**Precalentamiento.** Antes de la primera inferencia, ScopeWeaver llama a `/api/show`, rechaza un modelo que no declare `tools` y lo carga con una petición `/api/chat` vacía usando el `num_ctx` y `keep_alive` del perfil. Ocurre dentro del mismo hueco de concurrencia.

**Compactación.** En Ollama nativo con ventana de 64K o menos, la compactación se dispara al **72%** de la ventana y conserva pares completos de llamada/resultado. Si la compactación experimental noa está activada, esta protección tiene prioridad: noa se omite y se emite un aviso. Con 32K o menos, la parte de cada salida de herramienta que ve el modelo se corta a **16.384 bytes**; entre 32K y 64K, a 32.768 bytes. La salida completa se conserva en un archivo de derrame. Los formatos en la nube mantienen su comportamiento anterior.

**Juez de aprobación.** Un perfil de juez elegido explícitamente tiene prioridad. Si no, se prefiere el perfil nativo real usado por el agente de origen (incluidos binding, pool y failover); si el origen es nube o falta, se usa el perfil activo. Para un juez nativo, el tiempo de espera por defecto es **90 s** y la acción por fallo es **preguntar**. Las aprobaciones de usuario caducan a los 300 segundos con denegación por defecto. La salida nativa se restringe a un esquema JSON de decisión/comentario y se valida igualmente.

---

## Servidores genéricos compatibles con OpenAI (vLLM, LM Studio)

Usa los formatos existentes: elige **OpenAI (Chat Completions)** y apunta la URL base al endpoint `/v1` del servidor. **No** uses el formato nativo de Ollama para ellos.

- **Cargar modelos** en formato nativo lee `/api/tags`; si responde 404 o 405, cae a `/v1/models`, así que la lista de un servidor genérico puede aparecer.
- El chat **nunca** cambia en silencio: un perfil nativo siempre llama a `/api/chat`, que un servidor genérico no ofrece. El error de "modelo o endpoint no encontrado" y su sugerencia de `ollama pull` no aplican ahí; descargar un modelo no arregla un servidor vLLM o LM Studio. Cambia el formato a OpenAI.

---

## Cargar modelos y Probar conexión

**Cargar modelos** lee `/api/tags` y `/api/show`. Cada entrada muestra tamaño de parámetros, cuantización y distintivos `tools` / `thinking` / `vision`. Un modelo cuyos metadatos omiten `tools` muestra: *"Advertencia: este modelo no declara soporte nativo de llamadas a herramientas. Los agentes de ScopeWeaver requieren llamadas a herramientas para actualizar los grafos de activos y exploración."* Los metadatos fallidos se muestran como desconocidos, no como ausentes.

**Probar conexión** comprueba el modelo con `/api/show`, ejecuta una prueba de humo de una llamada a herramienta con el `num_ctx`, `keep_alive` y `num_predict` del perfil, y lee `/api/ps`. El panel muestra si el contexto asignado cubre la petición, si la llamada a herramienta volvió, el tiempo de carga en frío y la velocidad en tokens/s. Si falta el modelo, muestra el comando exacto, por ejemplo `ollama pull qwen2.5-coder:32b`. ScopeWeaver no descarga modelos ni crea Modelfiles.

---

## Diagnósticos y telemetría

**Probar conexión** hace una prueba de inferencia deliberada; elegir una plantilla solo descubre metadatos.

Los diagnósticos nativos muestran `size` y `size_vram` de `/api/ps` como porcentaje de memoria GPU. Tamaño total ausente o cero, o VRAM ausente, significa **desconocido**, no 0%. VRAM explícitamente cero con total positivo significa 0%. Es una estimación de memoria, no un reparto exacto de pesos ni una garantía de permanencia del modelo.

Un porcentaje igual o inferior al 80% avisa de que el trabajo en CPU puede ralentizar la generación. Otros modelos cargados y un juez explícito con distinto modelo o contexto en el mismo endpoint producen avisos de memoria y recarga por separado.

Los avisos de proximidad al límite de contexto son una heurística del registro del servidor basada en los tokens reportados, no un detector de truncamiento.

---

## Niveles de hardware

Puntos de partida aproximados por VRAM o memoria unificada de Apple. **Son estimaciones, no benchmarks.** El contexto indicado es la **Ventana de contexto (K)** del perfil y debe caber tanto en el límite del modelo como en tu memoria (pesos + caché KV, que crece con el contexto y `OLLAMA_NUM_PARALLEL`).

| Memoria | Modelo de ejemplo | Ventana (K) | Papel |
| --- | --- | --- | --- |
| 8–12 GB | `qwen2.5-coder:7b`, `qwen3:8b` | `16`–`32` | Juez de aprobación o perfil auxiliar; débil como agente principal |
| 16–24 GB | `qwen2.5-coder:14b`, `qwen3:14b` | `32` | Agente principal en estación de trabajo |
| 32–48 GB | `qwen2.5-coder:32b`, `qwen3:32b` | `32` | Agente principal. Usa `64` solo si el modelo lo admite; estos declaran 32K y 40K |
| 64–128 GB+ | `llama3.3:70b` | `64` (el modelo declara 128K) | Agente principal en hosts con mucha memoria |

---

## Modelos candidatos

Solo modelos cuya página de la [biblioteca de Ollama](https://ollama.com/library) declara la capacidad `tools` (comprobado el 2026-10-15). Tamaños de la cuantización por defecto.

| Modelo | Descarga | Notas |
| --- | --- | --- |
| `qwen2.5-coder:7b` | 4,7 GB | Candidato práctico más pequeño; también candidato a juez |
| `qwen2.5-coder:14b` | 9,0 GB | Candidato para estación de trabajo |
| `qwen2.5-coder:32b` | 20 GB | Candidato para host grande |
| `qwen3:8b`, `qwen3:14b`, `qwen3:32b` | 5,2 / 9,3 / 20 GB | También declaran `thinking` |
| `llama3.3:70b` | 43 GB | Hosts con mucha memoria |

`qwen3:72b` y `llama3.2:8b` no aparecen en la biblioteca; `llama3.2` se distribuye como 1B y 3B. Confirma `tools` con **Cargar modelos** en tu propio servidor.

**Limitación observada (2026-10-15):** declarar `tools` no garantiza llamadas reales. En pruebas de rama, `qwen2.5-coder:7b` escribió JSON con forma de herramienta como texto plano, así que ninguna herramienta se ejecutó, y `qwen3:4b` pasó el diagnóstico pero no completó tareas autónomas largas. Trata los modelos pequeños como no validados para el agente principal. Ver [verificación](../verificacion.md).

---

## Ollama puede reenviar a la nube

ScopeWeaver no puede ver qué hace el servidor Ollama después. **Ollama puede reenviar peticiones a modelos en la nube de Ollama.** El servicio Compose incluido fija `OLLAMA_NO_CLOUD=1`, pero un Ollama de host o remoto debe desactivar la nube por su cuenta (`OLLAMA_NO_CLOUD=1` o `disable_ollama_cloud` en `~/.ollama/server.json`).

Para bloquear salidas desde el lado de ScopeWeaver, ver [red y entorno](red-y-variables.md).

Si no tienes hardware para ejecutar un modelo en local, ver [**modelos gratuitos en línea**](modelos-gratuitos.md).
