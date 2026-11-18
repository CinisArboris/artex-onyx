# Plantillas de proveedores en la nube

Español · [← índice](../proveedores-llm.md)

Ruta en la UI: **LLM → Nuevo → Plantilla de configuración**.

Una plantilla solo rellena el formulario de un perfil **nuevo**. No guarda, no activa, no modifica perfiles existentes. Después de guardar, hay que pulsar **Activar** en la tarjeta.

---

## Tabla de referencia rápida

| Proveedor | Formato | URL base | Modelo de la plantilla | Esfuerzo | Campo de salida |
| --- | --- | --- | --- | --- | --- |
| DeepSeek | OpenAI Chat Completions | `https://api.deepseek.com` | `deepseek-flash` | `low` | `max_tokens` |
| Anthropic Claude | Anthropic | `https://api.anthropic.com` | `claude-haiku-5-5` | `low` | `max_tokens` |
| OpenAI | OpenAI Responses | `https://api.openai.com/v1` | `gpt-6-luna` | `low` | `max_output_tokens` |
| Alibaba Qwen (internacional) | OpenAI Chat Completions | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` | `qwen3.8-flash` | `low` | `max_completion_tokens` |
| Moonshot Kimi | OpenAI Chat Completions | `https://api.moonshot.ai/v1` | `kimi-k3` | `low` | `max_completion_tokens` |
| OpenRouter | OpenAI Chat Completions | `https://openrouter.ai/api/v1` | `deepseek/deepseek-v4.1-flash` | `low` | `max_tokens` |
| Z.ai GLM (API general) | OpenAI Chat Completions | `https://api.z.ai/api/paas/v4` | `glm-5.3` | `low` | `max_tokens` |
| OpenCode Go | OpenAI Chat Completions | `https://opencode.ai/zen/go/v1` | `glm-5.3-flash` | `max` | `max_tokens` |
| Cheaper Inference | OpenAI Chat Completions | `https://api.cheaperinference.com/v1` | `gpt-5.4-mini` | (omitido) | `max_completion_tokens` |

Todas las plantillas de nube usan límite de salida `16384` y ventana de contexto `0` (200K por defecto) salvo que la sección del proveedor diga otra cosa.

---

## Z.ai GLM-5.3

![Plantilla de perfil nuevo GLM-5.3](../../screenshots/en/llm-templates.png)

Vista previa de la plantilla sin clave API; no es una conexión real.

| Campo | API general (recomendada) | Coding Plan (referencia) |
| --- | --- | --- |
| Formato | OpenAI Chat Completions | OpenAI Chat Completions |
| URL base | `https://api.z.ai/api/paas/v4` | `https://api.z.ai/api/coding/paas/v4` |
| Modelo | `glm-5.3` | `glm-5.3` |
| Pensamiento | `enabled` | `enabled` |
| Esfuerzo de razonamiento | `low` | `low` |
| Ventana de contexto (K) | `0` (200K por defecto) | `0` (200K por defecto) |
| Límite de salida | `16384` | `16384` |

GLM-5.3 acepta texto, tiene ventana de 1M de tokens y exige razonamiento activado. Esfuerzos admitidos: `low`, `high`, `max`.

**Frontera del Coding Plan.** Z.ai limita Coding Plan a herramientas oficialmente compatibles. ScopeWeaver no está en esa lista y una plantilla de endpoint no establece permiso ni soporte. Obtén autorización de Z.ai antes de usarlo. Consulta [integración de herramientas](https://docs.z.ai/devpack/tool/others) y [política de uso](https://docs.z.ai/devpack/usage-policy).

Los endpoints y la facturación son distintos: el consumo de la API general es independiente de la cuota de suscripción. Verifica acceso al modelo y facturación antes de probar. Guía oficial: [GLM-5.3](https://docs.z.ai/guides/llm/glm-5.3).

Verificado contra documentación oficial el **2026-10-03**. Disponibilidad, derechos de cuenta y políticas pueden cambiar.

---

## OpenCode Go (pasarela de suscripción)

| Campo | Valor |
| --- | --- |
| Formato | OpenAI Chat Completions |
| URL base | `https://opencode.ai/zen/go/v1` |
| Modelo | `glm-5.3-flash` |
| Pensamiento | omitido (valor del modelo) |
| Esfuerzo de razonamiento | `max` |
| Ventana de contexto (K) | `0` (200K por defecto) |
| Límite de salida | `16384`, enviado como `max_tokens` |
| Cabecera de sesión | `x-opencode-session` (obligatoria; la plantilla la rellena) |

La pasarela es compatible con OpenAI. **Cada petición debe llevar la cabecera `x-opencode-session`**; sin ella responde `400 MissingSessionID`. ScopeWeaver envía sus propios ids de sesión como valor.

**Credencial.** La clave pertenece a tu suscripción de opencode, que la guarda en su propio archivo de autenticación: ScopeWeaver nunca lo lee. Pega la clave en el perfil o define `OPENCODE_API_KEY`. Revisa los términos de la suscripción antes de guardar: una plantilla no establece permiso.

```bash
SCOPEWEAVER_LLM_PROVIDER=opencode-go
SCOPEWEAVER_LLM_MODEL=glm-5.3-flash
# Clave: OPENCODE_API_KEY se usa si no hay clave genérica; SCOPEWEAVER_LLM_API_KEY la sustituye.
```

`opencode-go` resuelve al formato OpenAI Chat Completions. La plantilla usa `max` a petición del operador, así que espera el mayor consumo de tokens de razonamiento; bájalo si no lo necesitas. **Cargar modelos** lee el catálogo de la pasarela y el campo de modelo sigue editable para otros ids como `qwen3.7-plus`.

---

## Cheaper Inference (pasarela compatible con OpenAI)

| Campo | Valor |
| --- | --- |
| Formato | OpenAI Chat Completions |
| URL base | `https://api.cheaperinference.com/v1` |
| Modelo | `gpt-5.4-mini` |
| Pensamiento | omitido (valor del modelo) |
| Esfuerzo de razonamiento | omitido (valor del modelo) |
| Ventana de contexto (K) | `0` (200K por defecto) |
| Límite de salida | `16384`, enviado como `max_completion_tokens` |

Una clave da acceso a modelos de varios laboratorios. Límites del modelo comprobados en la lista pública `https://api.cheaperinference.com/public/models` el 2026-11-05: 400K de contexto, 128K de salida.

**Comportamiento de la clave.** Elegir esta plantilla, o cambiar la URL base hacia o desde la pasarela, **borra la clave ya escrita** en el formulario, para que una clave de otro endpoint nunca se envíe aquí y viceversa. Tras elegir la plantilla, pega tu clave; **Cargar modelos**, **Probar conexión** y guardar fallan con mensaje de clave ausente hasta entonces. Volver a elegir la plantilla o cambiar solo la ruta en la pasarela conserva la clave escrita.

Crea la clave en <https://cheaperinference.com> ([documentación](https://cheaperinference.com/docs)). La plantilla usa la ruta OpenAI Chat Completions ya existente y está cubierta por pruebas unitarias de plantilla, no por llamadas reales.

---

## Claude 5.5: pensamiento

Aplica solo a ids de la forma `claude-…-5-5` (o su instantánea `claude-…-5-5-YYYYMMDD`) en el host oficial (`https://api.anthropic.com` o URL base vacía).

| Ajuste | Qué se envía |
| --- | --- |
| Vacío (**Omitir**) | `thinking: {type: adaptive, display: summarized}` |
| **Adaptive** | Igual que Omitir |
| **On** (`enabled`) | `enabled` con `display: summarized` |
| **Off** (`disabled`) | `disabled` en turnos de agente |

- Sonnet y Opus 5.5 **rechazan** `disabled`: úsalos con **Omitir** o **Adaptive**. Haiku acepta `disabled` solo con esfuerzo `low`, `medium` o `high`, y la plantilla lo usa para ahorrar salida.
- En bucles de herramientas, los bloques de pensamiento se reenvían con sus llamadas, incluidos bloques con texto vacío y `redacted_thinking`. Se conservan en memoria para las últimas 128 llamadas (máximo 4 MiB por proveedor). Los bloques con texto resumido también se guardan en el historial; los vacíos y redactados no sobreviven a un reinicio a mitad de bucle.
- Las llamadas de resumen y del juez de aprobación ya no envían `temperature`, y descartan su `disabled` con Sonnet, Opus y Haiku en esfuerzo `xhigh` o `max`.
- Los endpoints Anthropic-compatibles de terceros (DeepSeek, Kimi, Z.ai) y otros ids no cambian: pensamiento vacío sigue omitiéndose.

**Verificado solo con fixtures de petición y stream**; el comportamiento real y la facturación de Claude no están verificados.

---

## Notas de fiabilidad

- El esfuerzo `low` es un **valor económico sin probar** fuera de GLM-5.3. Si las respuestas se cortan o son superficiales, sube hacia el valor del proveedor: DeepSeek `high`, GPT-6 Luna `medium`, Qwen `xhigh`, Kimi K3 `max`.
- OpenAI usa el formato Responses porque GPT-6 Luna admite herramientas en Chat Completions solo con esfuerzo `none`.
- OpenRouter: los modelos Claude necesitan marcadores de caché que el formato Chat Completions no puede enviar, por eso la plantilla usa DeepSeek Flash, que cachea prefijos automáticamente.
- Gemini no tiene plantilla: requiere la actualización a Norma SDK v0.4.4 (pendiente).

Para reducir coste, ver [tokens y costes](tokens-y-costes.md). Para configuración por entorno, ver [red y variables de entorno](red-y-variables.md).

Si no quieres pagar todavía, ver [**modelos gratuitos en línea**](modelos-gratuitos.md): el mismo procedimiento con planes gratis, y el criterio para saber si un modelo sirve como agente.
