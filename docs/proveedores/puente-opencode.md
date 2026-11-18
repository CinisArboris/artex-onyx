# Puente OpenCode SDK

Español · [← índice](../proveedores-llm.md)

> **No lo necesitas para una PoC.** El puente solo vale la pena si quieres el runtime nativo de OpenCode en lugar de la API HTTP. Además **no funciona en Windows** en ninguno de sus dos modos. Si estás en Windows, usa cualquiera de las [plantillas en la nube](nube.md) o [Ollama](inferencia-local.md).

Formato de perfil: `opencode-sdk`. Es un formato independiente, con credenciales propias. No sustituye las rutas de API ni OpenCode Go.

---

## Dos entornos fijados

| `SCOPEWEAVER_OPENCODE_RUNTIME` | Entorno | Frontera de entrada |
| --- | --- | --- |
| `native` (predeterminado) | CLI oficial y `@opencode/client` 2.3.0 | Conserva el JSON original de argumentos de herramienta, incluidos enteros de 64 bits, en respuestas e historial. Rechaza números inseguros fuera de los argumentos originales autoritativos. |
| `legacy` | SDK y CLI 1.19.4 | JSON válido y representable con seguridad; enteros de valor absoluto ≤ 9,007,199,254,740,991. Puede normalizar espacios y escapes HTML/Unicode equivalentes; no conserva los bytes léxicos originales. Rechaza enteros inseguros e historial de pensamiento opaco o firmado. |

`native` arranca el CLI auténtico con `serve --stdio` y `run --server --session`; el cliente oficial gestiona descubrimiento, eventos e interrupción. Cada llamada tiene rutas temporales privadas de home, configuración y datos. Las definiciones nativas reales siguen registradas: los permisos nativos son ASK y un guard público `execute.before` deniega toda herramienta que no sea del anfitrión antes de ejecutarse. Las capturas vuelven a Norma para ejecución y aprobación.

Los proveedores personalizados de `legacy` usan el mapeo explícito compatible con OpenAI del adaptador.

**Soporte de plataforma.** Ningún modo funciona en Windows: falta la limpieza del árbol de procesos POSIX. La verificación local cubre **macOS arm64**; CI ejecuta fixtures del entorno real en **Linux x64**. Otras plataformas no están verificadas. Versiones de SDK no probadas requieren verificación aparte.

---

## Puesta en marcha

Usa **Bun 1.3.14**; versiones superiores no están verificadas.

```bash
cd adapters/opencode-sdk
bun install --frozen-lockfile --ignore-scripts
export SCOPEWEAVER_OPENCODE_TOKEN="$(openssl rand -hex 32)"
bun run start
```

El token debe tener al menos **32 caracteres no blancos**. Guárdalo para el campo **Token del puente**. El listener por defecto es `127.0.0.1:4097`.

Después: **LLM → Nuevo → Plantilla de configuración → OpenCode SDK · puente**.

| Campo | Valor |
| --- | --- |
| Formato | `opencode-sdk` |
| URL del puente | `http://127.0.0.1:4097` |
| Modelo | `providerID/modelID`, con **Cargar modelos** o entrada explícita |
| Token del puente | El `SCOPEWEAVER_OPENCODE_TOKEN` del puente, **no** una clave de nube |
| Streaming | Activado |
| Pensamiento / esfuerzo | Omitidos por la plantilla; usa solo valores que el modelo admita |
| Límite de salida | `0` usa el valor del anfitrión; el puente limita a lo que declare el modelo |
| Proxy / cabecera de sesión | No admitidos; la plantilla borra ambos |

Para proveedores explícitos, usa un JSON dedicado con `SCOPEWEAVER_OPENCODE_PROVIDER_CONFIG`, siguiendo el esquema del adaptador. Nunca apuntes a un archivo de autenticación de usuario guardado.

---

## Comportamiento de la plantilla y de las credenciales

La plantilla borra una clave de nube sin guardar, deja el modelo vacío y **no** hace peticiones de modelo, inferencia, guardado ni activación. Introduce el token y pulsa **Cargar modelos** para leer el catálogo sin inferir.

Al editar:

- Un token vacío conserva el valor guardado **solo** para el mismo endpoint SDK.
- Cambiar la URL borra el token introducido.
- Cambiar endpoint o formato con token vacío no reutiliza la credencial guardada.
- `localhost`, `127.0.0.1` y `::1` son ámbitos de token **distintos**.

`SCOPEWEAVER_OPENCODE_TIMEOUT_MS` acepta 0–86.400.000 ms; `0` deja la duración bajo control de cancelación del anfitrión. Los cuerpos de petición están limitados a **16 MiB** y el exceso falla explícitamente.

---

## Red y cuentas

- El puente debe compartir espacio de nombres de red con ScopeWeaver. **Dentro de Docker, `localhost` es el propio contenedor.**
- El proveedor final puede seguir siendo un servicio en la nube, así que el modo **LLM solo local** **rechaza** los perfiles SDK.
- El puente no admite proxies: configura la conectividad del proveedor en el puente.
- Un id de modelo gratuito significa disponibilidad actual, no derecho de suscripción, permanencia ni uso ilimitado.
- El puente no elige alternativas de pago ni importa cuentas. El failover configurado en el anfitrión es independiente.

**Punto de partida gratuito verificado (2026-10-25 KST):** si aparece en **Cargar modelos**, elige `opencode/deepseek-v4.1-flash-free`. Una ejecución sin sesión iniciada en macOS arm64, con CLI nativo auténtico 2.1.4, pasó la prueba de conexión de la UI, una escritura Bash del anfitrión tras aprobación ASK manual, el contenido exacto `FREE_OK` más salto de línea y la respuesta final. Usó **3 POST de inferencia** con reintentos desactivados, sin alternativa de pago ni ejecución nativa observada. Esto demuestra un flujo pequeño y aislado de esa fecha, no disponibilidad permanente.

---

## Preparar una degradación

Este formato añade `opencode-sdk` a la restricción `CHECK` de formatos de perfil en la base de datos. **Un binario anterior no arranca mientras existan perfiles SDK.**

Antes de degradar: respalda la base y convierte esos perfiles a un formato admitido anterior o elimínalos usando la versión actual. La verificación previa usó bases desechables; no se migró una base de producción.

Ver también: [adaptador OpenCode SDK](../../adapters/opencode-sdk/README.es.md).
