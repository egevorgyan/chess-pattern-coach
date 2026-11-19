# Kickoff — Chess Pattern Coach (proyecto vitrina)

Nov 5, 2026 · @Ema

## Brief

Chess Pattern Coach es un backend en Java que analiza tus partidas de Lichess y Chess.com, detecta los errores que repetís y los explica en lenguaje natural con un LLM, verificando cada afirmación contra Stockfish. Su objetivo no es competir como producto: es demostrar a empresas, preferentemente del exterior, cómo integrás IA en un backend serio.

**Qué tiene que demostrar quien lea el repo:**

- Backend Java con Spring de nivel producción: capas claras, tests, CI, Docker.
- Integración de LLM con criterio: RAG, tool calling, un agente acotado y un MCP server propio.
- Evaluaciones: que la tasa de error de la IA se mide, no se supone.
- Proceso spec-driven: las specs y decisiones están versionadas en el repo.

**Tesis del README:** las explicaciones del LLM se verifican contra el motor, y esta es la tasa de error medida.

**Por qué ajedrez.** Stockfish da una respuesta correcta y objetiva para cada posición, así que cada frase del modelo se puede comprobar automáticamente. Casi ningún proyecto de RAG tiene una fuente de verdad así. Además, todo lo que existe en este nicho está en Python o TypeScript.

**Restricciones:** proyecto personal, por fuera del trabajo; sin sobreingeniería; repo público con README, demo y orden impecables; el deploy es opcional.

## Alcance del MVP

El MVP cubre el circuito completo de punta a punta para un solo usuario: importar, analizar, detectar patrones, explicar y medir. Todo lo demás queda afuera hasta que eso funcione.

| Entra en el MVP | Queda afuera |
| --- | --- |
| Importar partidas de Lichess y Chess.com por usuario, con APIs públicas | Login, cuentas y multiusuario |
| Análisis de cada jugada con Stockfish local y clasificación de errores | Frontend elaborado: alcanza con una página simple o Swagger |
| Detección de patrones entre partidas: aperturas débiles, tipos de error, fase de la partida | Puzzles, entrenamiento interactivo, planes semanales |
| Explicaciones en lenguaje natural ancladas en el análisis del motor | Fine-tuning o modelos propios |
| Chat con el coach que responde con datos de tus partidas (RAG) | Integraciones de pago, límites de uso |
| MCP server con herramientas de análisis y consulta de partidas | Escalado y alta disponibilidad |
| Set de evaluaciones con métricas publicadas en el README | Deploy público (opcional, al final) |
| Docker Compose que levanta todo con un comando |  |

**Criterio de terminado del MVP:** alguien clona el repo, corre un comando, importa sus partidas y recibe un reporte de patrones con explicaciones verificadas, y el README muestra las métricas de las evaluaciones.

## Arquitectura

Es un monolito modular en Spring Boot: un solo servicio desplegable con módulos separados por responsabilidad, más Stockfish y PostgreSQL como dependencias.

&#91;embedded content: arquitectura · ingesta, análisis, coach y verificación\]

- **Importador:** trae las partidas por usuario desde las APIs públicas de Lichess y Chess.com y las guarda en PGN.
- **Analizador:** corre Stockfish sobre cada jugada, clasifica errores (imprecisión, error, blunder) y agrega patrones entre partidas.
- **Coach:** arma el contexto con RAG sobre tus partidas y patrones, y usa Stockfish como herramienta cuando necesita evaluar una posición nueva.
- **Verificador:** antes de devolver una explicación, extrae las jugadas y evaluaciones que menciona y las compara con el motor; si no coinciden, la rechaza o la regenera.
- **API REST y MCP:** la misma lógica expuesta a una web simple y como MCP server para usarlo desde Claude Desktop.
- **Evals:** corren el coach sobre un set fijo de casos y miden la tasa de afirmaciones que el verificador rechaza.

## Stack y decisiones

Cada elección prioriza lo que un tech lead de una empresa Java reconoce y lo que se puede mantener sin sobreingeniería.

| Tema | Decisión | Por qué | Alternativa descartada |
| --- | --- | --- | --- |
| Lenguaje y framework | Java 21+ con Spring Boot | Es tu stack y lo que el proyecto tiene que mostrar | Python: es lo que usa todo el nicho, y te quita el diferencial |
| Integración con LLM | Spring AI | Trae ChatClient, tool calling, vector stores, evaluación y MCP dentro del ecosistema Spring | LangChain4j: válido, pero Spring AI encaja mejor con el resto |
| Modelo | API de Claude | Ya es tu elección; buen tool calling | Modelos locales: suman complejidad sin sumar al objetivo |
| Base de datos | PostgreSQL con pgvector | Una sola base para datos y embeddings; ya la usás | Base vectorial separada: un servicio más sin necesidad |
| Motor | Stockfish por UCI, en su propio contenedor | Determinístico, gratis y es la fuente de verdad del proyecto | Evaluación en la nube de Lichess: depende de un tercero y tiene límites |
| Arquitectura | Monolito modular | Un solo despliegue, módulos claros | Microservicios: sobreingeniería para un proyecto de una persona |
| Tests | JUnit 5 y Testcontainers | Tests de integración con Postgres y Stockfish reales | Mocks de todo: no prueban lo que importa |
| Infra local | Docker Compose | Levantar todo con un comando | Kubernetes: no aporta para esto |
| CI | GitHub Actions | Build, tests y evals en cada push |  |
| Interfaz | Página simple o Swagger UI | El foco es el backend | Frontend en React: tiempo que no suma al objetivo |

## Evaluaciones

Las evaluaciones son lo que separa este repo de un tutorial: miden si el coach dice la verdad sobre el tablero, y los números van publicados en el README.

**Set de casos.** Unas 50 posiciones fijas tomadas de partidas reales, cada una con su evaluación de Stockfish guardada como respuesta esperada. Se versionan en el repo para que cualquiera pueda reproducir los resultados.

| Métrica | Qué mide | Cómo se calcula |
| --- | --- | --- |
| Jugadas ilegales | Si el modelo menciona jugadas imposibles en la posición | Cada jugada citada se valida con una librería de reglas de ajedrez |
| Coincidencia con el motor | Si "la mejor jugada era X" coincide con Stockfish | Se compara la jugada citada con las mejores del motor a profundidad fija |
| Clasificación del error | Si "esto fue un blunder" coincide con la caída de evaluación real | Umbrales de centipeones definidos en la spec |
| Tasa de rechazo del verificador | Cuántos borradores se rechazan antes de llegar al usuario | Contador por corrida |
| Costo y latencia | Cuánto cuesta y tarda cada explicación | Tokens y tiempo registrados por llamada |

**Cómo se presenta.** Una tabla en el README que compara el coach sin verificador contra el coach con verificador. Esa diferencia es la tesis del proyecto hecha número.

**En CI.** Una corrida reducida de evals en cada push, para que un cambio de prompt que empeora la precisión se note antes de llegar a main.

## Estructura del repo y README

El repo tiene que leerse en cinco minutos: qué hace, cómo se levanta, cómo está armado y cuánto se equivoca.

```
chess-pattern-coach/
├── README.md
├── CLAUDE.md
├── docker-compose.yml
├── docs/
│   ├── specs/          # una spec por funcionalidad, versionada
│   ├── adr/            # decisiones de arquitectura (ADR)
│   └── diagrams/
├── src/main/java/.../
│   ├── ingestion/      # importador de Lichess y Chess.com
│   ├── analysis/       # Stockfish y detector de patrones
│   ├── coach/          # RAG, agente y prompts
│   ├── verification/   # verificador contra el motor
│   ├── api/            # REST
│   └── mcp/            # MCP server
├── evals/              # casos, runner y resultados
└── .github/workflows/  # CI con build, tests y evals
```

**Contenido del README, en este orden:**

1. Una línea con la tesis y un GIF de la demo.
2. Resultados de las evals: la tabla con y sin verificador.
3. Cómo levantarlo: un solo comando con Docker Compose.
4. Arquitectura: el diagrama y un párrafo por módulo.
5. Decisiones y trade-offs, con link a los ADR.
6. Cómo se construyó: el flujo spec-driven con IA y link a las specs.
7. Limitaciones conocidas y próximos pasos.

## Roadmap

Cinco hitos llevan al MVP y un sexto, opcional, lo publica. Cada hito termina con algo que funciona y se puede mostrar, no con código a medias.

&#91;embedded content: roadmap · 6 hitos, el MVP cierra en el 4\]

| Hito | Terminado cuando |
| --- | --- |
| 0. Specs y esqueleto | Brief, spec del MVP y primeros ADR en `docs/`; proyecto Spring Boot con CI en verde y Docker Compose con Postgres y Stockfish |
| 1. Ingesta | Un endpoint importa las partidas de un usuario de Lichess y de Chess.com y las guarda; tests de integración con Testcontainers |
| 2. Análisis y patrones | Cada jugada tiene evaluación y clasificación; un endpoint devuelve los patrones del usuario |
| 3. Coach y verificador | El coach explica errores y responde preguntas sobre tus partidas; el verificador rechaza afirmaciones que no coinciden con el motor |
| 4. MCP, evals y README | MCP server usable desde Claude Desktop; evals con métricas publicadas; README completo con GIF de demo |
| 5. Deploy (opcional) | Una instancia pública con datos de ejemplo y límite de uso |

## Riesgos y decisiones abiertas

**Riesgos**

| Riesgo | Mitigación |
| --- | --- |
| Analizar cientos de partidas con Stockfish lleva tiempo | Profundidad fija configurable, análisis asíncrono y caché por posición (FEN) |
| Costo de la API de Claude al correr evals seguido | Set reducido en CI y el completo a mano; registrar tokens por corrida |
| Las APIs de Lichess y Chess.com tienen límites de uso | Importación incremental y respeto de los límites documentados |
| El alcance crece (puzzles, frontend, multiusuario) | Todo lo que no está en la tabla del MVP va a un backlog, no al hito actual |
| El verificador solo cubre afirmaciones concretas | Documentarlo como limitación: verifica jugadas y evaluaciones, no ideas estratégicas generales |

**Decisiones abiertas para el hito 0**

- [ ] Nombre final del proyecto y del repo, que ya está creado.
- [ ] Versión de Java y de Spring AI a fijar.
- [ ] Profundidad de Stockfish para análisis y para verificación.
- [ ] Umbrales de centipeones para imprecisión, error y blunder.
- [ ] Qué embeddings usar para el RAG.
- [ ] Si la interfaz es Swagger UI o una página mínima.

## Semilla de CLAUDE.md

Para arrancar la sesión de desarrollo, copiá esto como `CLAUDE.md` en la raíz del repo y ajustalo después del hito 0.

```markdown
# Chess Pattern Coach

## What this is
A Java backend that imports a user's games from Lichess and Chess.com,
analyzes them with Stockfish, detects recurring mistakes, and explains
them with an LLM. Every claim the LLM makes about a position is verified
against Stockfish before it reaches the user. This is a public portfolio
project: code quality, tests and docs matter as much as features.

## Stack
- Java 21+, Spring Boot, Spring AI (Claude API)
- PostgreSQL + pgvector
- Stockfish via UCI, in its own container
- JUnit 5 + Testcontainers, GitHub Actions, Docker Compose

## Architecture
Modular monolith. Modules: ingestion, analysis, coach, verification,
api, mcp. Modules talk through interfaces, never through each other's
internals. No microservices.

## Workflow
- Spec-driven: every feature starts as a spec in docs/specs/ and is
  approved before code is written.
- Architecture decisions go in docs/adr/ as short ADRs.
- Work one milestone at a time (see docs/specs/roadmap.md). Anything
  outside the current milestone goes to the backlog.

## Rules
- Never let the LLM state a move or evaluation that the verifier has not
  checked against Stockfish.
- Every new endpoint has an integration test with Testcontainers.
- Changes to prompts must run the eval suite; do not merge if accuracy drops.
- Keep the README results table in sync with the latest eval run.
- No over-engineering: prefer the simplest design that meets the spec.

## Commands
- Start everything: docker compose up
- Tests: ./mvnw verify
- Evals: ./mvnw -pl evals exec:java (to be defined in milestone 4)
```
