# Red y variables de entorno

Español · [← índice](../proveedores-llm.md)

---

## Variables de entorno

`SCOPEWEAVER_LLM_*` sustituye a `ARTEX_LLM_*`; una variable moderna explícitamente vacía también sustituye a la antigua. Ambas tienen prioridad sobre el bloque `llm` de `config.json` (ruta desde `SCOPEWEAVER_CONFIG` o `ARTEX_CONFIG`).

| Sufijo | Significado |
| --- | --- |
| `PROVIDER` | `anthropic`, `openai`, `openai-responses`, `opencode-go` u `ollama` |
| `MODEL`, `BASE_URL`, `PROXY` | Modelo, endpoint y proxy LLM opcional |
| `API_KEY` | Obligatoria para Anthropic/OpenAI salvo URL base local; token opcional para Ollama |
| `CONTEXT_WINDOW_K`, `MAX_TOKENS`, `KEEP_ALIVE`, `MAX_CONCURRENT` | Controles de contexto, salida, permanencia y concurrencia |
| `THINKING_TYPE`, `REASONING_EFFORT`, `STREAM` | Opciones de razonamiento y transmisión |

Variables de política: `SCOPEWEAVER_LOCAL_LLM_ONLY` / `ARTEX_LOCAL_LLM_ONLY` y `SCOPEWEAVER_LOCAL_LLM_ALLOWED_CIDRS` / `ARTEX_LOCAL_LLM_ALLOWED_CIDRS` (separadas por comas).

---

## De dónde sale la credencial

Al arrancar, los formatos Anthropic y OpenAI usan `ANTHROPIC_API_KEY` o `OPENAI_API_KEY` cuando no hay clave genérica configurada, incluidos gateways locales autenticados.

| Proveedor | Variable |
| --- | --- |
| Anthropic | `ANTHROPIC_API_KEY` |
| OpenAI y compatibles | `OPENAI_API_KEY` |
| `opencode-go` | `OPENCODE_API_KEY` — **nunca** `OPENAI_API_KEY`. Se infiere cuando solo está definida esta. |
| Ollama nativo | `OLLAMA_API_KEY`, nunca las anteriores |

**Para arrancar un perfil local sin clave teniendo esas variables definidas**, fija explícitamente `SCOPEWEAVER_LLM_API_KEY=` (vacío). Eso anula las credenciales del archivo y suprime el respaldo a la clave del proveedor.

**Guardar un perfil nuevo con clave vacía no importa credenciales del entorno.** *Probar conexión* con clave vacía puede reutilizar la clave guardada de un perfil existente si se pasa su id. Sin id, una prueba local de Anthropic/OpenAI reutiliza la clave activa solo si formato, URL base normalizada y proxy coinciden con el servicio activo. Un endpoint local, formato o proxy distinto no hereda la clave; Ollama nativo nunca hereda una clave de nube activa.

---

## Ejemplo `.env` con Ollama en el host

```bash
# .env para ScopeWeaver en Docker con Ollama en el host
# Sustituye your-installed-tool-model por un tag con tools que muestre Cargar modelos.
SCOPEWEAVER_LLM_PROVIDER=ollama
SCOPEWEAVER_LLM_BASE_URL=http://host.docker.internal:11434
SCOPEWEAVER_LLM_MODEL=your-installed-tool-model
SCOPEWEAVER_LLM_CONTEXT_WINDOW_K=32
SCOPEWEAVER_LLM_MAX_CONCURRENT=1
SCOPEWEAVER_LLM_KEEP_ALIVE=30m
SCOPEWEAVER_LLM_MAX_TOKENS=4096
SCOPEWEAVER_LOCAL_LLM_ONLY=false   # añade el /32 o /128 real antes de activar el modo estricto
# SCOPEWEAVER_LOCAL_LLM_ALLOWED_CIDRS=192.168.1.20/32  # sustituye por la IP de inferencia resuelta
```

```json
{
  "llm": { "provider": "ollama", "base_url": "http://127.0.0.1:11434", "model": "your-installed-tool-model", "context_window_k": 32, "llm_max_concurrent": 1, "keep_alive": "30m", "max_tokens": 4096 },
  "local_llm_only": false,
  "local_llm_allowed_cidrs": []
}
```

---

<a id="política-de-red-local_llm_only"></a>
## Modo LLM solo local

Apagado por defecto; los proveedores en la nube siguen disponibles. Actívalo en **Ajustes del sistema → Red LLM solo local** o con `SCOPEWEAVER_LOCAL_LLM_ONLY=true`.

Cuando está activo:

- **El HTTP LLM** (inferencia, listas de modelos, pruebas de conexión, precalentamiento y juez de aprobación) permite **loopback más CIDR de confianza explícitos**. Las direcciones IPv4 privadas, ULA IPv6, LAN y Docker **ya no se confían implícitamente**. Añade solo direcciones de servidores de inferencia que controles, preferiblemente `/32` para IPv4 o `/128` para IPv6.
- **Siempre se rechazan** direcciones link-local, multicast y no especificadas, incluso dentro de un CIDR de confianza. Los nombres DNS se comprueban contra **todas** sus direcciones resueltas al abrir la conexión; todas deben estar permitidas. Se rechazan todas las redirecciones y los proxies de perfil LLM.
- **Se bloquean** la búsqueda web de agentes (en todos los backends), las pruebas de conectividad de búsqueda, la comprobación y la descarga de actualizaciones. La pantalla de actualización describe esta política aparte de un fallo de red. El estado de versión local, el progreso y la reversión siguen disponibles.
- Los cambios en caliente aplican sin reiniciar, incluidas sesiones ya creadas. Una descarga admitida antes de activar el modo estricto se detiene en la siguiente operación de red protegida y elimina el staging parcial; lo ya enviado no se puede recuperar.
- Los CIDR de `SCOPEWEAVER_LOCAL_LLM_ALLOWED_CIDRS` (o `ARTEX_LOCAL_LLM_ALLOWED_CIDRS`) también aplican, no se copian a la lista guardada y no se pueden editar en la UI. Un `true` por entorno es un **suelo** que la UI no puede desactivar.
- El tráfico de objetivos, las herramientas de objetivo y el proxy global siguen conectados.

**Esto es una política de aplicación, no una garantía criptográfica ni de host completamente desconectado.** Para una barrera real, usa cortafuegos o proxy obligatorio.

---

## Migrar perfiles estrictos de LAN o Docker

Un `http://127.0.0.1:11434` en el mismo equipo no necesita CIDR. Un endpoint LAN como `http://192.168.1.20:11434` necesita `192.168.1.20/32`; una dirección IPv6 como `fd00::20` necesita `fd00::20/128`.

En Docker, resuelve el nombre **desde el contenedor de ScopeWeaver**, porque host y contenedor pueden ver direcciones distintas:

```bash
docker compose exec scopeweaver getent ahosts host.docker.internal
docker compose exec scopeweaver getent ahosts ollama
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{.GlobalIPv6Address}}{{end}}' "$(docker compose ps -q ollama)"
```

Si la imagen no tiene `getent`, revisa el mapeo de pasarela con `docker compose exec scopeweaver cat /etc/hosts` e inspecciona la dirección del contenedor Ollama con `docker inspect`.

Usa la IP de inferencia que ScopeWeaver **realmente** alcanza, no un ejemplo copiado ni una subred Docker entera. Ni `host.docker.internal` ni el nombre `ollama` otorgan confianza implícita. Vuelve a comprobar cuando Docker recree el servicio o cambie su dirección.

---

## Resumen de decisiones de red

| Situación | Qué hacer |
| --- | --- |
| Ollama en el mismo equipo, ScopeWeaver nativo | `http://127.0.0.1:11434`, sin CIDR |
| Ollama en el host, ScopeWeaver en Docker | `http://host.docker.internal:11434`; en modo estricto, añade el `/32` resuelto |
| Ollama como servicio Compose | `http://ollama:11434`; el IP del contenedor cambia si Docker lo recrea |
| vLLM o LM Studio | Formato **OpenAI Chat Completions**, URL `/v1`. No uses formato nativo de Ollama |
| Quieres bloquear toda salida a la nube | Modo solo local + `OLLAMA_NO_CLOUD=1` en el servidor Ollama |

Para la configuración de cada proveedor, vuelve al [índice](../proveedores-llm.md).
