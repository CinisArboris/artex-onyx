# Proveedores LLM: índice

Español · [← documentación](indice.md)

Esta página es el punto de entrada. Cada tema vive en su propio documento para que no tengas que leer 450 líneas cuando solo necesitas un dato.

| Documento | Cuándo abrirlo |
| --- | --- |
| [**Modelos gratuitos en línea**](proveedores/modelos-gratuitos.md) | No tienes hardware y quieres probar con planes gratuitos: Groq, OpenRouter, Cerebras, Google AI Studio… |
| [Plantillas de proveedores en la nube](proveedores/nube.md) | Vas a crear un perfil Anthropic, OpenAI, DeepSeek, Qwen, Kimi, OpenRouter, GLM, OpenCode Go o Cheaper Inference |
| [Puente OpenCode SDK](proveedores/puente-opencode.md) | Quieres usar el runtime de OpenCode en lugar de la API HTTP |
| [Inferencia local con Ollama](proveedores/inferencia-local.md) | Quieres cero dependencia de la nube, o tienes GPU propia |
| [Tokens y costes](proveedores/tokens-y-costes.md) | La factura subió, o quieres que no suba |
| [Red y variables de entorno](proveedores/red-y-variables.md) | Necesitas `local_llm_only`, configurar por `.env` o diagnosticar conectividad |

---

## Conceptos que aplican a todos los proveedores

**Formatos disponibles.** `anthropic`, `openai` (Chat Completions), `openai-responses` (Responses API), `ollama` (nativo) y `opencode-sdk` (puente). Los cuatro primeros conservan URL base propia, proxy, cabecera de sesión, reintentos, límite de frecuencia y concurrencia, así que pasarelas y servidores compatibles con OpenAI funcionan sin cambios.

**Qué hace una plantilla.** Rellena el formulario de un **perfil nuevo**. No guarda, no activa, no modifica perfiles existentes y no hace peticiones por sí sola. Después de aplicar una, el formato, la URL base y el modelo siguen editables.

**Lo que una plantilla NO te da.** Un perfil nuevo **no se activa solo**. Hay que pulsar **Activar** en la tarjeta, y entonces el motor cambia en caliente sin reiniciar.

**Guardar no es conectar.** *Probar conexión* hace una llamada real al modelo (no se registra en `llm_usage`). *Cargar modelos* solo consulta el catálogo. Solo la primera demuestra que hay inferencia.

**Un perfil puede ser rechazado.** Con **LLM solo local** activo, cualquier perfil en la nube se rechaza. Ver [red y entorno](proveedores/red-y-variables.md).

---

## Enlaces antiguos

Estas secciones existen solo para que los enlaces de otros documentos no se rompan. Cada una apunta ya a su documento.

<a id="opencode-sdk-bridge"></a>
### Puente OpenCode SDK
Movido a **[Puente OpenCode SDK](proveedores/puente-opencode.md)**.

<a id="ollama-nativo-inferencia-local"></a>
### Ollama nativo (inferencia local)
Movido a **[Inferencia local con Ollama](proveedores/inferencia-local.md)**.

<a id="controlar-el-uso-de-tokens"></a>
### Controlar el uso de tokens
Movido a **[Tokens y costes](proveedores/tokens-y-costes.md)**.

<a id="entorno-y-configuración"></a>
### Entorno y configuración
Movido a **[Red y variables de entorno](proveedores/red-y-variables.md)**.

---

Las restricciones de uso del [README](../README.es.md) siguen aplicando: ejecuta ScopeWeaver solo en un entorno aislado localmente, sea cual sea el proveedor que elijas. Ver también: [agentes](agentes.md).
