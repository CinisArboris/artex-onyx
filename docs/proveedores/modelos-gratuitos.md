# Modelos gratuitos en línea

Español · [← índice](../indice.md)

Esta guía resuelve el caso de no tener hardware: usar modelos abiertos por API gratis. Casi todos se conectan igual, porque hablan el dialecto **OpenAI Chat Completions**, y ScopeWeaver ya lo soporta.

> **Lo que esta guía no puede darte: una lista de claves gratuitas.** Los planes gratuitos cambian de nombre, de límite y a veces desaparecen cada pocas semanas. Darte una lista fija sería mentirte. Lo que sí es estable es **el procedimiento** y **el criterio para saber si un modelo sirve o no**. La sección de proveedores se marca con fecha de verificación para que sepas qué caducó.

---

## 1. La prueba que decide todo

Antes de gastar tiempo en un proveedor, entiende esto:

**ScopeWeaver no es un chat. Es un agente.** Para funcionar necesita que el modelo **llame herramientas** (`insert_assets`, `add_finding`, `Bash`, `Read`…). Un modelo que solo sabe conversar no sirve: razonará, escribirá un párrafo precioso y morirá sin ejecutar nada.

**Probar conexión no detecta esto.** Verificado en `agent/provider.go:624`: esa prueba envía un único turno pidiendo la palabra "OK" y comprueba que llegue texto. No hay herramientas en juego. Un modelo sin tool calling **pasa** la prueba y luego falla en el primer chat real.

Entonces el orden correcto es:

| Paso | Qué demuestra |
| --- | --- |
| 1. Probar conexión | El endpoint, la clave y el id del modelo existen |
| 2. Una conversación de una línea | El modelo responde por el camino del agente |
| 3. **Pedirle algo que requiera una herramienta** | **Tool calling: la prueba real** |
| 4. Una tarea completa pequeña | Sirve para trabajo autónomo |

El paso 3 es el filtro. Si falla, no importa lo grande que sea el modelo.

---

## 2. Conectar un proveedor compatible con OpenAI

Sirve para Groq, Cerebras, Mistral, Together, DeepInfra, Fireworks, OpenRouter, GitHub Models, Google AI Studio (modo compatible), vLLM propio y casi cualquier pasarela.

**LLM → Nuevo** y rellena:

| Campo | Qué poner | Nota |
| --- | --- | --- |
| Nombre | Lo que quieras | Solo etiqueta |
| Formato | **OpenAI (Chat Completions)** | No uses "Responses" salvo que el proveedor lo documente |
| URL base | La del proveedor, normalmente acabada en `/v1` | Ejemplo: `https://api.groq.com/openai/v1` |
| Modelo | El id exacto, ej. `llama-3.3-70b-versatile` | Se puede escribir a mano o usar **Cargar modelos** |
| Clave API | La del plan gratuito | Obligatoria salvo endpoint local (ver abajo) |
| Ventana de contexto (K) | `0` o `32` | Ver sección 4 |
| Límite de salida | `4096` para empezar | `16384` si el modelo lo soporta |
| Peticiones concurrentes | `1` en planes gratuitos | Casi todos limitan por minuto |

**Cargar modelos** consulta `/v1/models` y te da la lista real de tu cuenta. Es la forma de no adivinar ids. No guarda ni activa nada.

Cuando esté, **Guardar** y luego **Activar** en la tarjeta. Un perfil nuevo no se activa solo.

### La clave es obligatoria si el endpoint no es local

Regla exacta del código (`agent/provider.go:355`): si la clave está vacía y el host **no** resuelve a loopback o dirección privada, ScopeWeaver rechaza el perfil con *«llm: API key required for non-local compatible endpoint»*.

Consecuencia práctica: **un proveedor gratuito en internet siempre necesita una clave**, aunque el plan sea gratis. "Gratis" significa que no pagas, no que no te registres. Los planes gratuitos piden cuenta y clave precisamente para limitarte por cuota.

La única excepción real es un endpoint en tu propia máquina o red local (vLLM, LM Studio, Ollama), donde la clave puede ir vacía.

---

## 3. Ajustes para planes gratuitos

Los planes gratuitos no fallan por calidad, fallan por **cuota**. Estos ajustes evitan que el agente queme la cuota en los primeros minutos:

| Ajuste | Por qué |
| --- | --- |
| **Peticiones concurrentes = 1** | Por defecto hay 3 agentes de trabajo en paralelo. Cada uno gasta una llamada. Con 1 evitas ráfagas que disparan el límite por minuto. |
| **Ventana de contexto baja (`32`)** | El contexto no se envía al proveedor en la nube, solo controla cuándo se compacta el historial. Bajarlo hace que el agente resuma antes y cada llamada lleve menos tokens. |
| **Límite de salida `4096`** | Un tope alto no acorta respuestas, corta las largas. Pero en cuota gratuita un límite bajo evita que una respuesta rara consuma todo el cupo. |
| **Esfuerzo de razonamiento vacío** | Muchos proveedores gratuitos rechazan el campo o lo ignoran. Si ves un error 400 mencionando `reasoning_effort`, déjalo vacío. |
| **Pensamiento vacío** | Igual. Deja que el modelo use su valor por defecto. |
| **Retries bajos** | Un reintento reenvía la petición completa y consume cuota otra vez. En `-1` desactivas esa capa. |

También puedes bajar el número de agentes de trabajo en **Sistema → Ajustes**. Para una PoC, 1 o 2 es suficiente y multiplica la duración de tu cuota.

### Si el proveedor limita por minuto

Los campos **Peticiones por segundo** y **Peticiones por minuto** del perfil existen para eso. Si el plan gratuito te da, por ejemplo, 30 peticiones/minuto, ponlo ahí y ScopeWeaver espacia las llamadas en lugar de fallar.

---

## 4. Cómo saber si el modelo sirve para agente

Antes de confiar en él, comprueba estas tres cosas. La primera descarta el 80%.

**1. ¿Soporta tool calling?**
Busca en la documentación del proveedor "function calling" o "tools". Si no aparece, no sirve para ScopeWeaver, punto.

**2. ¿La ventana de contexto alcanza?**
Los prompts de ScopeWeaver son largos: system prompt, esquemas de herramientas y los resultados de las anteriores. Por debajo de **32K** el agente se pasa la vida compactando. Ideal 128K o más.

**3. ¿Aguanta un bucle de 10-20 turnos?**
Algunos modelos gratuitos pequeños se pierden en la mitad y dejan de llamar herramientas. Es lo que mide la sección 5.

### La prueba práctica

Crea una conversación con un agente que tenga herramientas atadas (el builtin `pentest` ya las trae) y pídele algo así:

> Lista los activos registrados de esta conversación usando `list_assets`.

| Resultado | Diagnóstico |
| --- | --- |
| Ejecuta la herramienta y responde con el resultado | Tool calling funciona. Sirve. |
| Escribe "no tengo acceso a herramientas" o describe qué haría | **No sirve.** Está simulando. |
| Escribe un JSON de llamada como texto plano en vez de ejecutarla | Capa de compatibilidad rota. **No sirve.** |

El tercer caso está documentado en este mismo repo con `qwen2.5-coder:7b` en local (ver [inferencia local](inferencia-local.md)). Es un fallo del modelo, no de ScopeWeaver.

---

## 5. Proveedores con plan gratuito

Comprobado en **noviembre de 2026**. Los planes gratuitos cambian rápido: verifica el límite actual en la web del proveedor antes de confiar en una PoC.

| Proveedor | URL base | Por qué mirarlo | Ojo con |
| --- | --- | --- | --- |
| **Groq** | `https://api.groq.com/openai/v1` | El más rápido del mercado. Plan gratuito real, ideal para probar el bucle de agente sin esperas. Modelos Llama y Qwen. | Límites por minuto ajustados. El tamaño del modelo importa: usa los grandes para agente. |
| **Cerebras** | `https://api.cerebras.ai/v1` | Velocidad muy alta, plan gratuito para desarrollo. | Catálogo más pequeño. |
| **Google AI Studio** | `https://generativelanguage.googleapis.com/v1beta/openai/` | Plan gratuito generoso. Endpoint compatible con OpenAI (la ruta `/openai/` es la clave). Modelos Gemini. | La compatibilidad es una capa encima; prueba tool calling antes de confiar. |
| **Mistral** | `https://api.mistral.ai/v1` | Nivel gratuito para experimentación. Modelos abiertos. | El nivel gratuito tiene límites estrictos por minuto. |
| **OpenRouter** | `https://openrouter.ai/api/v1` | Un solo registro da acceso a muchos modelos, **con modelos marcados `:free`**. La vía más rápida para probar varios sin crear cuenta en cada uno. | Los `:free` tienen cola y límite diario. Un modelo gratuito puede agotarse y cambiar a otro sin avisar. |
| **GitHub Models** | `https://models.github.ai/inference` | Gratis con cuenta de GitHub, sin tarjeta. Muy útil para empezar. | Cuota diaria baja y pensada para pruebas, no para tareas largas. |
| **Together / DeepInfra / Fireworks** | Cada uno la suya en `/v1` | Suelen dar crédito de bienvenida en lugar de plan perpetuo. | Es crédito, no gratis permanente. Se acaba. |
| **Z.ai GLM** | `https://api.z.ai/api/paas/v4` | GLM tiene plan gratuito por tiempo limitado según promoción. Ver [plantillas en la nube](nube.md). | La plantilla Coding Plan **no** está autorizada para ScopeWeaver. Usa la API general. |

Sobre **"oxigen"**: no existe un proveedor con ese nombre en el ecosistema de LLM. Probablemente recuerdas **Oxygen** de algún hosting, o **Oxlo**, o es una confusión con **OpenRouter**. Si tienes el enlace, dime cuál es y lo verifico.

### Ruta recomendada si empiezas de cero

1. **OpenRouter con un modelo `:free`** — un registro, muchos modelos, y puedes cambiar sin crear cuentas nuevas.
2. Si el modelo gratuito te decepciona, **Groq** con un modelo grande: la velocidad hace que el bucle se sienta vivo.
3. Cuando la PoC funcione y quieras estabilidad, pasa a un plan de pago barato. Un modelo gratuito con cola no sirve para demostrar nada a un equipo.

---

## 6. OpenRouter en detalle

Es la vía más práctica para tu caso, así que merece detalle.

**Alta.** Cuenta en <https://openrouter.ai>, clave en *Keys*. No pide tarjeta para modelos `:free`.

**En el formulario:**

| Campo | Valor |
| --- | --- |
| Formato | OpenAI (Chat Completions) |
| URL base | `https://openrouter.ai/api/v1` |
| Modelo | Un id con sufijo `:free`, ej. `deepseek/deepseek-r1:free` |
| Clave API | Empieza por `sk-or-v1-...` |

**Cargar modelos** te da el catálogo completo. Los `:free` llevan el sufijo en el propio id. Filtra por ahí.

**Tres cosas que debes saber:**

- Los modelos `:free` tienen **cola compartida**. A horas punta la latencia sube mucho. No es un fallo de ScopeWeaver.
- Hay **límite diario** de peticiones en el nivel gratuito. Si lo agotas, los errores son 429.
- OpenRouter puede **enrutar a otro proveedor** del mismo modelo. Si un `:free` deja de responder, el modelo se retiró.

Si quieres cabeceras de atribución (opcionales), OpenRouter las documenta como `HTTP-Referer` y `X-Title`. ScopeWeaver tiene el campo **Cabecera de sesión** para una sola cabecera; para dos, no hay campo. No es obligatorio para funcionar.

---

## 7. Cuando algo falla

| Error | Causa real | Solución |
| --- | --- | --- |
| `llm: API key required for non-local compatible endpoint` | Clave vacía en endpoint remoto | Introduce la clave. No hay atajo. |
| `401` / `Unauthorized` | Clave incorrecta o caducada | Regenera la clave |
| `404 model not found` | Id mal escrito o el modelo no existe en tu cuenta | Usa **Cargar modelos** en lugar de escribir a mano |
| `400` mencionando `reasoning_effort` o `thinking` | El proveedor no acepta ese campo | Vacía **Esfuerzo de razonamiento** y **Pensamiento** |
| `429 Too Many Requests` | Cuota agotada | Baja concurrentes a 1, pon límite por minuto, o espera |
| Probar conexión pasa pero el chat no hace nada | **Falta tool calling** | Sección 4. Cambia de modelo. |
| El agente se detiene a mitad de tarea | Modelo pequeño o contexto insuficiente | Sube a un modelo mayor o baja la ventana de contexto |
| Respuestas cortadas | Límite de salida bajo | Sube **Límite de salida** |

Para ver el error real del proveedor, activa **Grabaciones LLM** antes de la conversación: guarda la petición y la respuesta completas.

---

## 8. Qué esperar de un modelo gratuito

Sé honesto contigo mismo sobre el alcance. Un modelo gratuito en línea te sirve para:

- Demostrar el flujo completo: alcance → agente → herramienta → hallazgo → informe.
- Aprender el producto sin gastar.
- Validar que tu caso de uso tiene sentido.

**No** te sirve para:

- Tareas autónomas largas (20+ turnos) con fiabilidad.
- Trabajo real sobre un objetivo de producción.
- Comparar calidad entre modelos con seriedad.

La documentación del repo es explícita en esto: la fiabilidad de tareas autónomas no está validada para modelos pequeños ([verificación](../verificacion.md)). Úsalo como banco de pruebas, no como herramienta de producción.

Cuando la PoC convenza y necesites resultados repetibles, el paso natural es un plan de pago barato con un modelo grande, o un equipo con GPU para [inferencia local](inferencia-local.md).

---

## 9. Ficha para copiar

```
Proveedor:        ____________________
URL base:         ____________________
Modelo:           ____________________
¿Tool calling?:   [ ] sí  [ ] no   ← si es "no", descarta el modelo
Ventana contexto: ______ K
Límite salida:    _________ tokens
Concurrentes:     1
Límite por minuto: ________________
Prueba de herramienta: [ ] pasa  [ ] falla
```

Las restricciones de uso siguen aplicando: solo entornos aislados o con autorización explícita. Ver [licencia y exención de responsabilidad](../../README.es.md#licencia-y-exención-de-responsabilidad).
