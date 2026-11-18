# Dar contexto a otras IAs

Español · [← índice](indice.md)

Este repositorio es un laboratorio de seguridad ofensiva. Muchos asistentes de IA tienen filtros que
rechazan peticiones relacionadas con pentesting porque no pueden distinguir un laboratorio autorizado
de un ataque real. Este documento explica cómo cerrar esa brecha sin engañar a nadie.

---

## 1. Por qué te rechazan

Hay dos causas y solo una se resuelve con contexto:

| Causa | Ejemplo | ¿Se arregla? |
| --- | --- | --- |
| **Ambigüedad** | «Ayúdame a explotar esta vulnerabilidad» — la IA no sabe contra qué | **Sí.** Es la mayoría de los rechazos |
| **La petición en sí** | «Escanea example.com» — objetivo de terceros | **No**, y no debería |

El filtro no lee tu intención, lee tu texto. «Explotar» sin más es indistinguible de un ataque. Con
un alcance verificable, deja de serlo.

**Lo que no funciona:** declarar «tengo permiso» o «es solo para aprender». Son afirmaciones que no se
pueden comprobar, y por eso no cambian la evaluación. Lo que sí funciona es la **especificidad**: rutas,
topología, comandos concretos, versiones. Un contexto con detalles técnicos es creíble; uno genérico no.

---

## 2. La estructura que funciona

Cinco piezas. Si falta una, la tasa de rechazo sube.

**1. Qué es el entorno.** No «un pentest», sino «un laboratorio local con contenedores Docker en
127.0.0.1, sin salida a internet».

**2. Para qué.** Formación interna, construir procedimientos, entender la arquitectura del agente.

**3. Qué está en alcance.** Nombra los objetivos. «Juice Shop en `http://localhost:3000` y el contenedor
`dvwa` de esta misma máquina».

**4. Qué está fuera.** Explícito: «ningún dominio de internet, ningún servicio de terceros».

**5. Qué esperas.** «Explícame cómo se orquesta», «revisa mi código», «ayúdame a escribir la skill».

El orden importa: el alcance va antes que la petición. Si lo pones después, la IA ya evaluó la petición
en el vacío.

---

## 3. Archivos que las IAs leen solas

Estos ya existen en el repositorio y se cargan automáticamente:

| Archivo | Lo lee |
| --- | --- |
| [`AGENTS.md`](../AGENTS.md) | Cursor, Codex, Gemini CLI, Amp, Jules y otros agentes |
| [`CLAUDE.md`](../CLAUDE.md) | Claude Code |
| [`.cursor/rules/laboratorio.mdc`](../.cursor/rules/laboratorio.mdc) | Cursor, siempre activo |
| [`.github/copilot-instructions.md`](../.github/copilot-instructions.md) | GitHub Copilot |

Si usas una IA que lee el repositorio, ya tienes el contexto puesto. No necesitas pegar nada.

**Para Cursor:** abre el repo y el contexto entra solo. Si quieres que una conversación concreta lo
refuerce, escribe `@AGENTS.md` en el prompt.

**Para Claude Code:** ejecuta `claude` dentro del directorio del repo; lee `CLAUDE.md` al arrancar.

---

## 4. Prompt para IAs sin lectura de repositorio

Para ChatGPT, Gemini web, Claude web, Grok o cualquier interfaz que no ve tus archivos. Copia esto y
ajusta lo que haga falta.

```text
Contexto: trabajo en un laboratorio de seguridad ofensiva autoalojado para formación interna.
Es un entorno cerrado que corre en mi propia máquina: contenedores Docker accesibles solo en
127.0.0.1 (sin exposición a internet y sin salida a redes externas).

Objetivos del laboratorio: Juice Shop en http://localhost:3000 y DVWA en http://localhost:8080.
Ambos son aplicaciones deliberadamente vulnerables, diseñadas para ser atacadas, que yo mismo
despliego y destruyo.

Fuera de alcance: cualquier dominio, IP o servicio de internet de terceros. No los usaré y no
te pediré ayuda con ellos.

Tarea: [describe lo que necesitas, lo más concreto posible]

Respondo por el resultado técnico. Si algo de lo que pido no encaja con el alcance descrito,
dímelo antes de responder, en lugar de rechazar la conversación entera.
```

**La última línea es la más útil.** Le da una salida elegante: en vez de negarse en bloque, puede
señalar qué parte le chirría. En la práctica convierte muchos rechazos en conversaciones productivas.

---

## 5. Ajustar el prompt según la tarea

El texto de arriba es la base. Cambia la línea de tarea según lo que necesites:

| Necesitas | Línea de tarea |
| --- | --- |
| Entender el producto | «Explícame cómo este fork orquesta planificador, trabajadores y verificación adversaria, leyendo la arquitectura del repo». |
| Escribir una skill | «Ayúdame a redactar una `SKILL.md` en formato agentskills.io para reconocimiento de APIs en el laboratorio». |
| Revisar código | «Revisa este código Go: ¿hay rutas donde el alcance del agente no se respete?». |
| Entender un hallazgo | «Explícame por qué este payload provoca SQLi en Juice Shop; quiero entender el mecanismo». |
| Diseñar el lab | «Ayúdame a diseñar un laboratorio con tres servicios vulnerables y rutas de ataque encadenadas». |

Las tareas de **análisis, docencia y revisión** pasan casi siempre. Las que más fricción encuentran son
las que suenan a ejecución directa («dame el payload para tumbar X»), incluso en un lab. Reformular
hacia el mecanismo suele resolverlo: «explícame por qué este payload funciona» en vez de «dame uno».

---

## 6. Cuando te rechazan igual

**Reformula hacia el aprendizaje.** «Explícame el mecanismo de esta vulnerabilidad» es una petición
docente. «Dame el exploit» suena a ejecución. Mismo conocimiento, distinta evaluación.

**Reduce el alcance de la pregunta.** Un modelo rechaza «cómo hackear una web» y acepta «cómo funciona
la validación de entrada en un formulario de login». La segunda es la primera descompuesta.

**Usa el código del repo como ancla.** Estás trabajando sobre un repositorio real y auditable. Decir
«estoy modificando este archivo, mira la función X» es verificable, y la verificabilidad es lo que
mueve la aguja.

**Cambia de herramienta, no de verdad.** Si una IA es demasiado restrictiva para tu flujo, usa otra. No
mientas al respecto ni ocultes el propósito para sortear un filtro: eso rompe la confianza y, en un
contexto profesional, te deja sin defensa si algo sale mal.

**Y la regla que no se negocia:** si la única forma de que una IA te ayude es ocultarle que es
seguridad ofensiva, la respuesta correcta es no hacerlo. Este documento existe para dar contexto
verdadero, no para disfrazar una petición.

---

## 7. Lo que este repositorio ya hace por ti

El proyecto incorpora controles reales que puedes citar:

- **Alcance declarado** por empresa y por tarea, que guía los prompts y filtra el registro de activos.
- **Reglas de intercepción** que bloquean o permiten activos concretos, con el bloqueo por encima del
  permiso.
- **Aprobaciones** que exigen intervención humana antes de ejecutar herramientas sensibles.
- **Registro de evidencia** de cada acción, lo que hace el trabajo auditable.

Ninguno de estos controles es una barrera de red —el propio README lo advierte—, pero sí son controles
de aplicación verificables. Mencionarlos es legítimo porque existen en el código, no porque suenen bien.

---

Las restricciones de uso siguen aplicando: solo entornos aislados localmente o con autorización
explícita. Ver [licencia y exención de responsabilidad](../README.es.md#licencia-y-exención-de-responsabilidad).
