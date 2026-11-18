# Tokens y costes

Español · [← índice](../proveedores-llm.md)

Cada turno de agente **reenvía** el prompt del sistema, los esquemas de herramientas y toda la conversación. La entrada crece turno a turno hasta que la compactación acorta el historial. El coste no está en una llamada grande, sino en muchas llamadas que repiten lo mismo.

---

## Estado de la evidencia

| Dato | Fuente | Fiabilidad |
| --- | --- | --- |
| Efecto de `low` vs `max` en tokens y créditos | Medición real con GLM-5.3 en Coding Plan | Medido |
| Números de DeepSeek | GLM como aproximación | **No verificado** |
| DeepSeek, Claude, OpenAI, Qwen, Kimi, OpenRouter, OpenCode Go | Fixtures construidos desde la documentación de cada proveedor | No son llamadas reales |
| Comportamiento de Claude 5.5 | Fixtures de petición y stream | Sin clave de Claude |

GLM tiene tokenizador, contabilidad de caché y precio distintos a DeepSeek, así que **los números de DeepSeek están sin verificar**.

---

## Ajustes recomendados

1. Empieza por la plantilla económica de tu proveedor y deja la ventana de contexto en `0` salvo que una tarea necesite más historial.
2. Mantén un límite de salida positivo: `0` **no** omite el campo.
3. Mantén el esfuerzo en `low`; súbelo solo cuando los resultados empeoren.
4. Deja el juez de aprobación, la búsqueda web y la compresión noa apagados salvo que los necesites (todos vienen así).
5. Para tareas pequeñas, considera menos agentes de trabajo que los 3 por defecto. El efecto no está medido.
6. Si no necesitas un informe detallado por hallazgo, desactiva el disparador del agente **Redactor de informes** (*Sistema → Agente*).
7. Tras la primera tarea, revisa el uso por carril.

---

## Efecto medido

Una tarea pequeña: tres rutas en un objetivo local de prueba con un secreto sin autenticar plantado. 3 ejecuciones por perfil, cada una en una base nueva, dejando terminar las conversaciones del redactor. Los créditos usan los multiplicadores publicados de GLM-5.3 — `(entrada sin caché × 6,9 + entrada cacheada × 1,7 + salida × 24) / 10.000` — y son **calculados, no leídos de una factura**.

| Perfil | Tokens mediana (entrada + salida) | Créditos mediana | Problema plantado encontrado |
| --- | --- | --- | --- |
| Base: plantilla anterior (esfuerzo `max`, contexto `1000`, salida `0`) | 209.233 (172.576–216.452) | 75,1 (64,3–88,6) | 3 de 3 |
| Económico: esfuerzo `low`, salida `16384`, contexto `0` | 96.023 (95.060–105.023) | 26,6 (25,0–27,4) | 3 de 3, cada uno reportado por el redactor |

**Resultado:** 54% menos tokens y 65% menos créditos en la mediana; toda ejecución económica quedó por debajo de la base más barata. La llamada individual más grande usó 13.287 tokens de entrada y 3.829 de salida, así que no se alcanzó ni la ventana de 200K ni el límite de salida: **el ahorro viene del esfuerzo `low`**. El límite de salida y la ventana acotan tareas mayores y no se midieron.

Dispersión entre ejecuciones: 1,25× para la base, 1,10× para la económica.

**Reparto por carril** (créditos): base — trabajador 29,6%, planificador 29,3%, redactor 29,1%, preparación de objetivos 12,0%. Económico — redactor 38,8%, planificador 34,1%, trabajador 20,2%, preparación 6,9%. No hubo compactación en esas ejecuciones.

**Caché y razonamiento:** entrada cacheada 85,2% en la base y 92,3% en la económica. El razonamiento, incluido en la salida, fue 39,3% de los créditos en la base y 23,4% en la económica. GLM-5.3 siempre piensa, así que estas ejecuciones **no** miden el efecto de apagarlo.

---

## Los cuatro ajustes que deciden el coste

| Ajuste | Comportamiento en la nube | Guía |
| --- | --- | --- |
| **Ventana de contexto (K)** | Solo presupuesto local de compactación; no se envía al proveedor. `0` = 200K; los valores se acotan a 32–1000. | Una ventana mayor retrasa la compactación, así que cada llamada lleva más historial. A 200K, los resultados de herramienta antiguos empiezan a limpiarse por encima de ~115K tokens y el historial se resume hacia ~179K. |
| **Límite de salida** | Tope por respuesta. **`0` no se omite**: los turnos de agente envían 32768, las llamadas directas a Anthropic 8192. | Un tope no acorta las respuestas; corta las largas. Demasiado bajo trunca. Las plantillas usan 16384. |
| **Pensamiento** | `thinking.type`; vacío lo omite, salvo en Claude 5.5 oficial, donde vacío envía `adaptive`. El razonamiento se factura como salida. | DeepSeek piensa por defecto y documenta `disabled`. Con herramientas, DeepSeek reenvía razonamiento anterior como entrada. GLM-5.3 siempre piensa. |
| **Esfuerzo de razonamiento** | `low` a `max`; vacío lo omite. | Las plantillas usan `low`. Solo se midió en GLM-5.3; en otros proveedores es un valor económico sin probar. Los valores admitidos varían. |

---

## Optimización de contexto (automática)

- **Compactación de conversación** (integrada, activada): cerca de la ventana, los mensajes antiguos se resumen y se conservan los 8 últimos. Es una llamada extra; su salida se limita al límite de salida, con máximo 20.000.
- **Limpieza de salidas antiguas:** por encima del 60% de la ventana, los resultados de `Read`, `Bash`, `Grep`, `Glob`, `WebFetch`, `WebSearch`, `Edit` y `Write` más antiguos que los cinco últimos se sustituyen. Los resultados de grafo, evidencia y herramientas personalizadas aguantan hasta el siguiente resumen.
- **Tope de salida por herramienta:** cada bloque de texto de un resultado se corta a 30.000 bytes; la salida completa queda en disco con un puntero.
- **Compactación de grafo:** las partes inactivas del grafo de exploración se pliegan en resúmenes, una llamada por grupo elegible (tope 1500).
- **Compresión noa** (experimental, apagada) sustituye la compactación integrada.
- El disparador al 72% y los topes de 16/32 KiB aplican **solo** a Ollama nativo.

---

## Caché de prompt

- **Claude:** ScopeWeaver marca siempre el prompt del sistema, y los esquemas de herramientas anteriores a él, como cacheables en sesiones de planificador, trabajador, agente principal y chat (incluido el redactor). Cualquier otra petición en formato Anthropic con prompt de sistema y sin frontera marca todo el prompt: preparación de objetivos, resúmenes de conversación y grafo, preparación de preguntas paralelas y juez de aprobación. Un prompt más corto que el mínimo cacheable del modelo no se cachea aunque tenga marcador. El primer uso de cada prompt distinto escribe la caché a 1,25× del precio de entrada; las llamadas dentro de la vida de la caché leen al precio cacheado. Los prompts del trabajador cambian por intención, así que cada intención escribe una vez. **La colocación de marcadores está verificada por pruebas; el ahorro real no** (sin clave de Claude).
- **DeepSeek, Kimi y Z.ai** cachean prefijos automáticamente. DeepSeek y Kimi documentan que sus endpoints compatibles con Anthropic ignoran los marcadores de bloque.
- **OpenAI, Qwen y OpenRouter** (DeepSeek Flash) también cachean prefijos coincidentes automáticamente.
- Las pruebas confirman que turnos consecutivos en una sesión mantienen el mismo prompt y esquemas, y solo añaden historial, en los tres formatos de nube. La compactación, el cierre, las herramientas diferidas recién desbloqueadas y un `{{.Now}}` en el prompt cambian el prefijo.

---

## Qué multiplica las llamadas

- **Sesiones largas:** cada turno reenvía todo el historial.
- **Planificador:** sesión nueva en cada desperta (inicio, fin de intención, hallazgo nuevo, latido en reposo), con varios turnos cada una.
- **Descomposición de objetivos:** corre antes de planificar, hasta 8 turnos.
- **Redactor de informes:** agente con disparador activo en cada `report_finding`. Cada hallazgo abre su propia conversación, hasta 5 en paralelo.
- **Búsqueda web de DeepSeek:** cada búsqueda es una llamada aparte con la clave del perfil activo (tope de salida 1024, hasta 3 búsquedas por llamada). Se registran en `llm_usage` con carril `web_search`. **Una respuesta sin bloque de uso se registra con 0 tokens: la llamada cuenta, sus tokens son desconocidos.** Verifica en la consola de DeepSeek.
- **Juez de aprobación** (apagado): una llamada por acción de herramienta sin regla coincidente.
- **Preguntas paralelas:** solo a petición, hasta 12 llamadas de resumen antes de responder.
- **Probar conexión:** llamada real (tope 8192) que **no** se registra.
- **Reintentos:** reenvían la petición completa. Por defecto: 3 de conexión, 2 de respuesta vacía (solo formato OpenAI), 2 de ventana segura en el mismo proveedor, 2 reejecuciones tras error de modelo y hasta 2 continuaciones tras una respuesta que solo contiene razonamiento. `-1` desactiva una capa.
- **Cierre (wrap-up):** al agotarse turnos o tiempo, añade hasta 10 turnos más.

---

## Dónde mirar el consumo

- **Tareas → tarea → Resumen → Uso de tokens LLM (por modelo):** llamadas, entrada, salida, **razonamiento (dentro de salida)** y lectura de caché por modelo, desde la tabla `llm_usage`, siempre activa.
- **Panel** y **Reglas de intercepción → Uso de tokens de aprobación** también muestran razonamiento.
- **Grabaciones LLM:** una fila por llamada con carril, tokens, latencia y petición/respuesta en crudo. Apagado por defecto; activa **Grabación** antes de la tarea. Guarda prompts completos.

```sql
SELECT COALESCE(worker, '(none)') AS carril, model, COUNT(*) AS llamadas,
       SUM(input_tokens) AS entrada, SUM(cache_read) AS lectura_cache,
       SUM(cache_write) AS escritura_cache, SUM(output_tokens) AS salida,
       SUM(reasoning_tokens) AS razonamiento
FROM llm_usage
WHERE ts >= '2026-10-16 09:00' AND ts < '2026-10-16 10:00'  -- rango de tu tarea
GROUP BY 1, 2
ORDER BY SUM(input_tokens) + SUM(output_tokens) DESC;
```

Carriles (`worker`): `planner`, `worker`, `goals`, `compactor`, `judge`, `btw` (preguntas paralelas), `mainagent`, `web_search`. Vacío para conversaciones del redactor, reintentos y agentes personalizados.

**No sumes dos veces:** `cache_read` y `cache_write` ya están dentro de `input`, y `reasoning` ya está dentro de `output`.

Lecturas de caché se cuentan desde `prompt_tokens_details.cached_tokens`, `prompt_cache_hit_tokens` de DeepSeek y `cached_tokens` de Kimi. Escrituras se registran para OpenAI Chat, OpenAI Responses, Kimi K3, OpenRouter, Qwen y Claude. **Estos mapeos están verificados solo con fixtures.**

### Errores de contabilidad corregidos

- Los streams en formato Anthropic sobrecontaban la salida en `llm_usage`: un stream acumulativo de 10 y 20 se registraba como 30 en vez de 20. Solo inflaba las cifras de ScopeWeaver, no la factura del proveedor.
- Los totales internos de sesión y transcripción tenían el mismo error; ahora cuentan 20.
- Las preguntas paralelas en perfiles OpenAI Responses se reportaban como interrumpidas por falta de evento de parada; ahora `response.completed` cierra la respuesta normalmente.

### Lagunas conocidas

- El razonamiento es 0 cuando el proveedor no reporta cifra separada; **un 0 no prueba que el modelo no razonó**.
- Los nombres de campo de uso de la búsqueda de DeepSeek no están documentados.
- Gemini no tiene plantilla.
- El caché de historial de Claude (marcador en el último mensaje) necesita soporte del SDK.
- Los modelos de OpenRouter que solo respetan `reasoning.effort` unificado pueden ignorar el `reasoning_effort` de la plantilla.
