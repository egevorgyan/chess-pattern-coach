# MVP Spec — Chess Pattern Coach

Status: approved · Version: 1.0 · Date: 2026-11-19
Source: [`docs/kickoff.md`](../kickoff.md) plus the spec interview.

---

## 1. Goal and user

### Goal
Chess Pattern Coach is a Java backend that imports a player's games from Lichess and Chess.com and analyzes every move with Stockfish. It then detects the mistakes the player repeats and explains them in natural language with an LLM. **Every concrete claim the LLM makes about a position is checked against the engine before it reaches the user.**

The project is a public portfolio piece. Its purpose is to show how to integrate AI into a production-grade Java backend:

- RAG, tool calling, a bounded agent, and an MCP server of its own;
- **measured** error rates, not assumed ones;
- a spec-driven process, versioned in the repo.

**README thesis:** *LLM explanations are verified against the engine, and this is the measured error rate.*

### User
- **Primary user: a single chess player running the project locally.** They play blitz, rapid or classical on Lichess and/or Chess.com and want to know which mistakes they repeat and why. There is no login. The person who runs the instance owns all of its data.
- **Secondary audience: a technical reviewer**, such as a tech lead or recruiter at a Java company. They clone the repo, run one command, and judge the code, the architecture, and the published eval metrics.

### MVP done criterion
Someone clones the repo and runs `docker compose up`. They create a profile with their handles and import their games. They then get a pattern report with verified explanations, and the README shows the eval metrics comparing the coach without and with the verifier.

---

## 2. Glossary and domain parameters

| Term | Definition |
| --- | --- |
| **Profile** | A local entity that groups **one Lichess handle and/or one Chess.com handle** belonging to the same person. Patterns, report and chat are computed over all of the profile's games. |
| **Handle** | A username on a source platform (`lichess` or `chesscom`). |
| **Profile move** | A move played by the profile's side in a game. |
| **Win%** | The side to move's probability of winning, derived from the engine evaluation with the Lichess formula: `win% = 50 + 50 · (2 / (1 + e^(−0.00368208 · cp)) − 1)`. A mate for the side to move is 100, a mate against it is 0. |
| **Win% drop** | The mover's win% before the move (evaluated at its best move) minus the mover's win% after the played move. Negative values count as 0. |
| **Classification** | Each profile move gets a class from its win% drop: **inaccuracy** ≥ 10, **mistake** ≥ 20, **blunder** ≥ 30. Anything lower is unclassified, i.e. a "good" move. |
| **Game phase** | **Opening:** moves 1–12. **Endgame:** total non-pawn material on the board ≤ 13 points (N=B=3, R=5, Q=9, both sides added together). **Middlegame:** everything else. Endgame takes priority over opening. |
| **Low clock** | The mover's remaining clock is < 10% of the initial time. This needs `%clk` annotations in the PGN; games without them are excluded from this pattern. |
| **Claim** | A concrete, typed statement inside an LLM response: `best_move`, `move_class`, `eval` or `line`. |
| **Verified response** | An LLM response in which every claim has passed the verifier (§ 4, US-07). |

### Parameters (defaults, configurable through properties or env vars)

| Parameter | Default |
| --- | --- |
| Games imported per handle (N) | 100 most recent |
| Imported time controls | blitz, rapid, classical |
| Analysis depth | 16 |
| Verification and eval depth | 20 |
| Stockfish threads | 1 (determinism) |
| Minimum occurrences for a pattern | 5 |
| "Low clock" threshold | < 10% of initial time |
| "Winning position" threshold (squandered advantage) | win% ≥ 80 |
| Best-move tolerance | engine top-3 **and** ≤ 10 win% points from the best move |
| Eval-claim tolerance | same sign, and ≤ 10 win% points from the engine evaluation |
| Mate horizon for tactical patterns | mate in ≤ 5 |
| Max regenerations per response | 2 (3 attempts total) |
| Max tool calls per chat answer | 5 |
| Chat memory window | last 20 messages |
| Representative mistakes per pattern in the report | up to 3 |

---

## 3. Milestones

Each milestone ends with something that works and can be shown, never with half-done code.

| Milestone | Scope | Done when |
| --- | --- | --- |
| **M1 — Data pipeline (no LLM)** | Profile, import from Lichess and Chess.com, async Stockfish analysis, classification, patterns as data, Docker Compose, CI (build + tests) | US-01 … US-06 pass their acceptance criteria and CI is green |
| **M2 — Verifier, explanations and evals** | Structured LLM output, verifier, mistake explanations, pattern report with verified explanations, eval set, independent checker | US-07 … US-09 pass, and an eval run produces metrics without vs. with the verifier |
| **M3 — Chat agent and MCP** | Bounded chat agent (RAG, tools, memory) and an MCP server, both built on the same verified tools layer | US-10 and US-11 pass, and the MCP server works from Claude Desktop |
| **M4 — Publication** | README results table, reduced evals in CI, demo page and GIF, complete README | US-12 and US-13 pass; the MVP done criterion (§ 1) is met |

---

## 4. User stories and acceptance criteria

> Endpoint paths are indicative. The final contract lives in the OpenAPI document, which Swagger UI exposes. Every endpoint has an integration test with Testcontainers.

### US-01 — Create a profile · **M1**
*As a player, I want to create a profile with my Lichess and/or Chess.com handle so that all my games are analyzed together.*

- **AC1.** Given a Lichess handle, a Chess.com handle, or both, when I create a profile (`POST /api/profiles`), then the system checks that each handle exists on its platform, stores the profile, and returns its id and handles.
- **AC2.** Given a request with no handle, when I create a profile, then I get `400` with a message saying at least one handle is required.
- **AC3.** Given a handle that does not exist on its platform, when I create the profile, then I get `404`, the error names the platform and the handle, and no profile is created.
- **AC4.** Given an existing profile, when I fetch it (`GET /api/profiles/{id}`), then I see its handles, the number of imported games per source, and its latest analysis status.
- **AC5.** A profile has at most one handle per platform. Trying to add a second one returns `409`.

### US-02 — Import games · **M1**
*As a player, I want to import my recent games so that they can be analyzed.*

- **AC1.** Given a profile, when I trigger an import (`POST /api/profiles/{id}/imports`), then the system fetches up to the N most recent games per handle from the public APIs and stores each game's PGN and metadata: source, source id, date, colors, result, time control, ECO, and opening name when available.
- **AC2.** Only standard chess in blitz, rapid or classical is imported. Chess960, other variants, bullet, correspondence and daily games are skipped and reported as `skipped` in the import summary.
- **AC3.** Re-importing is incremental and idempotent. Games already stored (same source + source id) are not duplicated, and the summary reports `imported`, `alreadyPresent` and `skipped` counts.
- **AC4.** API calls follow the platforms' documented guidance: sequential requests, plus backoff and retry on HTTP `429`. If the import still cannot finish, it ends `FAILED` with a clear reason, and the games already stored are kept.
- **AC5.** A successful import automatically starts an analysis job for the new games (US-03).
- **AC6.** An integration test covers both sources against recorded HTTP fixtures. No test calls the live APIs.

### US-03 — Track analysis status · **M1**
*As a player, I want to see the progress of the analysis so that I know when my report is ready.*

- **AC1.** Analysis runs asynchronously. Triggering an import returns `202` with a job id, without waiting for the analysis.
- **AC2.** Given a job, when I query it (`GET /api/jobs/{id}`), then I see its state (`QUEUED`, `RUNNING`, `COMPLETED` or `FAILED`), games analyzed out of the total, and start and end timestamps.
- **AC3.** If one game fails to analyze (for example, an unparseable PGN), that game is marked as failed with a reason, and the job continues with the rest.
- **AC4.** Positions are cached by FEN and depth. An already-evaluated position is not sent to Stockfish again, and the job reports how many evaluations it served from cache.
- **AC5.** With `threads=1` and a fixed depth, analyzing the same game twice gives the same classifications.

### US-04 — View classified moves of a game · **M1**
*As a player, I want to see each of my moves with its evaluation and class so that I understand where a game went wrong.*

- **AC1.** Given an analyzed game, when I fetch it (`GET /api/games/{id}`), then every position has the engine evaluation (cp or mate) and best move at the analysis depth.
- **AC2.** Every profile move carries its win% before and after, its win% drop, its class (if any), its phase, and its remaining clock (if available).
- **AC3.** Opponent moves are evaluated but not classified.
- **AC4.** Classification follows exactly the thresholds in § 2. A unit test covers each class boundary, including mate-to-no-mate transitions.

### US-05 — Get detected patterns as data · **M1**
*As a player, I want to see the mistakes I repeat across games so that I know what to work on.*

- **AC1.** Given a profile with completed analysis, when I fetch its patterns (`GET /api/profiles/{id}/patterns`), then I get **only** the pattern types below, and only those with at least 5 occurrences:

  | Pattern type | Unit of occurrence | Detected when |
  | --- | --- | --- |
  | `WEAK_OPENING` | game, grouped by ECO + color | The profile's score in that opening (win = 1, draw = 0.5) is lower than its overall score with that color, **or** the average win% drop in the opening phase is higher than the profile's average |
  | `PHASE_WEAKNESS` | classified move, grouped by phase | The share of inaccuracies, mistakes and blunders in that phase is higher than the profile's overall rate |
  | `LOW_CLOCK_ERRORS` | classified move under low clock | The mistake/blunder rate under low clock is higher than the rate with enough time |
  | `HANGING_PIECE` | classified move (mistake or blunder) | Within the first 3 plies of the engine PV after the played move, the opponent wins ≥ 2 points of net material |
  | `MISSED_MATE` | profile move | Before the move the engine had a mate in ≤ 5 for the profile, and after the played move it no longer does |
  | `ALLOWED_MATE` | profile move | Before the move there was no forced mate against the profile, and after it the engine reports a mate in ≤ 5 against the profile |
  | `COLOR_WEAKNESS` | game, grouped by color | The score or average win% drop with one color is worse than with the other |
  | `SQUANDERED_ADVANTAGE` | game | The profile reached win% ≥ 80 and the game ended in a draw or a loss |
  | `TIME_CONTROL_WEAKNESS` | game, grouped by time control | The score or average win% drop in that time control is worse than the profile's average |

- **AC2.** Every pattern includes its type, its group key (ECO, phase, color or time control, when applicable), the occurrence count, metrics (rate, average win% drop, score), and references to the games and moves behind it.
- **AC3.** Patterns are sorted by severity, defined as occurrences × average win% drop. For game-level patterns, the drop is replaced by the score gap.
- **AC4.** This endpoint involves **no LLM**. It returns deterministic data.
- **AC5.** Every pattern type has a unit test with a hand-built fixture that triggers it and one that does not.

### US-06 — Run everything with one command · **M1**
*As a reviewer, I want to start the whole system with one command so that I can try it in minutes.*

- **AC1.** Given a clean clone with Docker installed, when I run `docker compose up`, then the app (with Stockfish inside its image) and PostgreSQL with pgvector start, and the app becomes healthy without any manual step.
- **AC2.** In M1, no API key is needed. From M2 on, the only requirement is `ANTHROPIC_API_KEY` in a `.env` file, documented in `.env.example`. Without the key, the M1 features still work, and the LLM features return a clear `503` error.
- **AC3.** Swagger UI is reachable at a documented URL.
- **AC4.** CI (GitHub Actions) runs `./mvnw verify` on every push and PR, with real Postgres and Stockfish through Testcontainers.

### US-07 — Get a verified explanation of a mistake · **M2**
*As a player, I want a natural-language explanation of a specific mistake so that I understand why it was bad and what was better. Every concrete statement in it must be true.*

- **AC1.** Given a classified profile move, when I request its explanation (`POST /api/moves/{id}/explanation`), then I get English text plus the list of claims it contains, all marked verified.
- **AC2.** The LLM returns structured output: the text plus typed claims (`best_move`, `move_class`, `eval`, `line`). The verifier rejects the draft if any of these hold:
  - a cited move is illegal in its position (rules library);
  - a `best_move` is not in the engine's top-3 at the verification depth, or is more than 10 win% points from the best move;
  - a `move_class` differs from the class computed in § 2;
  - an `eval` has the wrong sign, or is more than 10 win% points from the engine evaluation;
  - a `line` contains an illegal move, or its first move fails the best-move rule;
  - the text contains a move in SAN that is not declared as a claim.
- **AC3.** A rejected draft is regenerated, and the concrete verifier errors are fed back to the model. There are at most 2 regenerations.
- **AC4.** If the third attempt also fails, the response holds **only engine data** (played move, best move, win% drop, class) with `explanation: unavailable`. No unverified LLM text ever reaches the user.
- **AC5.** Every LLM call logs model, input and output tokens, latency, attempt number and verifier result.
- **AC6.** The verifier is tested with fixed drafts covering each rejection rule and a fully valid draft. These tests do not call the LLM.

### US-08 — Get the pattern report with verified explanations · **M2**
*As a player, I want a report of my patterns explained in plain language so that I get actionable advice.*

- **AC1.** Given a profile with completed analysis, when I request its report (`GET /api/profiles/{id}/report`), then I get every pattern from US-05. Each comes with a natural-language summary and up to 3 representative mistakes, each with a verified explanation as in US-07.
- **AC2.** Representative mistakes are the occurrences with the largest win% drop in that pattern.
- **AC3.** Pattern summaries go through the same verifier. Their claims are limited to moves and evaluations from the referenced games.
- **AC4.** A report is cached per profile and analysis version. Requesting it again without new games makes no new LLM calls.
- **AC5.** If analysis is not complete, the endpoint returns `409` with the job status.

### US-09 — Run the eval set with the independent checker · **M2**
*As a reviewer, I want reproducible metrics on how often the coach is wrong, with and without the verifier, so that the thesis is backed by numbers.*

- **AC1.** The eval set holds about 50 positions taken from public Lichess games (CC0), versioned under `evals/cases/`. It mixes phases and mistake types. Each case stores the FEN, the move played, and the expected answer at depth 20: engine top-3 with evaluations, win% drop, and class.
- **AC2.** The task in each case is the product task: "explain the mistake" (the same prompt and pipeline as US-07).
- **AC3.** The runner (`./mvnw verify -Pevals`) runs every case in two modes: **without verifier** (first draft returned as is) and **with verifier** (the full US-07 pipeline).
- **AC4.** An **independent checker** computes the metrics by comparing outputs against the stored expected answers, not by calling the live verifier:
  - % of responses citing an illegal move;
  - % of best-move claims that agree with the engine;
  - % of correct class claims;
  - errors in drafts vs. errors in final output;
  - verifier rejection rate;
  - % `explanation unavailable`;
  - average and p95 tokens, latency and estimated cost per explanation.
- **AC5.** Each run writes a machine-readable result (JSON) and a Markdown table to `evals/results/`, stamped with date, commit, model and prompt version.
- **AC6.** Expected answers can be regenerated from the FENs by a documented command. Regenerating them twice gives the same result.

### US-10 — Chat with the coach · **M3**
*As a player, I want to ask questions about my games in natural language and get answers grounded in my own data.*

- **AC1.** Given a profile, when I send a message (`POST /api/profiles/{id}/chat` with an optional `conversationId`), then I get an English answer and the `conversationId`.
- **AC2.** The chat is a **bounded agent** with read-only tools:
  - RAG search over the profile's classified mistakes and patterns;
  - exact queries for stats and patterns;
  - evaluate a position with Stockfish;
  - validate a move.

  It makes at most 5 tool calls per answer. Past the limit, it answers with what it has and says so.
- **AC3.** The RAG index holds one document per classified mistake (FEN, played move, best move, win% drop, class, opening, phase, game reference) and one per pattern, all filtered by profile. Numeric questions such as counts and rates are answered through exact queries, not similarity search.
- **AC4.** Every answer goes through the US-07 verification pipeline, with the same regeneration and fallback rules. An answer without concrete claims passes through unchanged.
- **AC5.** Memory is kept per `conversationId` in PostgreSQL, as a window of the last 20 messages. A follow-up question such as "and with black?" is answered in context.
- **AC6.** If the profile has no analyzed data relevant to the question, the coach says so instead of inventing an answer.
- **AC7.** Tests use a mocked LLM to cover tool routing, the tool-call limit, memory and the verification fallback.

### US-11 — Use the coach from Claude Desktop via MCP · **M3**
*As a player using Claude Desktop, I want to query my games and get verified explanations without leaving my client.*

- **AC1.** The app exposes an MCP server over streamable HTTP on a documented local URL. The README includes a ready-to-paste Claude Desktop configuration.
- **AC2.** The MVP tools are `list_profiles`, `list_patterns`, `get_game`, `explain_mistake` and `evaluate_position`. They reuse the same verified tools layer as the chat.
- **AC3.** Tools return engine data and **already-verified** explanations (`explain_mistake` runs the full US-07 pipeline).
- **AC4.** The README and the tool descriptions state that text Claude Desktop writes on its own, outside tool results, is not verified by this system.
- **AC5.** An integration test calls each tool through an MCP client and checks the response schema.

### US-12 — See eval results in the README; reduced evals in CI · **M4**
*As a reviewer, I want to see the verifier's impact at a glance, and know that regressions are caught.*

- **AC1.** The README shows the results table from the latest full eval run, without vs. with the verifier, with the date, commit and model of that run.
- **AC2.** A GitHub Actions workflow runs a reduced subset (about 10 fixed cases) on push to `main` and through `workflow_dispatch`. It uses an `ANTHROPIC_API_KEY` secret and fails if engine agreement or the illegal-move rate gets worse than a threshold stored in the repo.
- **AC3.** PR and fork builds do not need the secret; they run build and tests only.
- **AC4.** The full set runs by hand, and the README documents how to run it.

### US-13 — See a demo of the full flow · **M4**
*As a reviewer, I want to see the system working without installing it.*

- **AC1.** A minimal static page served by the app (no frontend framework) lets you enter handles, follow import and analysis progress, read the report, and chat.
- **AC2.** The README opens with the one-line thesis and a GIF of the full flow: profile → import → report → chat → Claude Desktop via MCP.
- **AC3.** The README follows the order in the kickoff:
  1. thesis and GIF;
  2. eval results;
  3. how to run;
  4. architecture;
  5. decisions with links to the ADRs;
  6. how the project was built, spec-driven;
  7. known limitations and next steps.

---

## 5. Non-functional requirements

- **Code quality:** modular monolith with these modules: `ingestion`, `analysis`, `coach`, `verification`, `api`, `mcp`. Modules talk through interfaces, never through each other's internals.
- **Testing:**
  - JUnit 5 and Testcontainers, with real PostgreSQL and real Stockfish;
  - no live calls to Lichess, Chess.com or Claude in the default test suite;
  - every endpoint has an integration test.
- **Determinism:** Stockfish runs with fixed depth and `threads=1`. Classifications and expected eval answers are reproducible.
- **Observability:** structured logs for every LLM call (tokens, latency, attempts, verifier outcome) and every analysis job.
- **Configuration and secrets:** all parameters in § 2 can be set through properties or env vars. No secret is ever committed.
- **Performance:** no hard target in the MVP. Analysis time per 100 games and cost per explanation are **measured and reported** in the README.
- **Language:** the UI, the coach's responses and the README are in English.

---

## 6. Out of MVP scope

Everything below goes to a backlog, not to the current milestone.

- Login, accounts and multi-user support.
- Chess960 and other variants.
- Bullet, correspondence and daily games.
- Tactical motifs beyond hanging piece, missed mate and allowed mate (forks, pins, skewers, and so on).
- Opening theory or any external knowledge base in the RAG.
- Puzzles, interactive training and weekly plans.
- An elaborate frontend (React or similar).
- Fine-tuning or custom models.
- Payments and usage limits.
- Scaling and high availability.
- Multi-language responses.
- Verification of general strategic ideas. The verifier covers moves, classes, evaluations and lines only; this is documented as a known limitation.

---

## 7. Constraints and preliminary decisions

These are technical decisions taken during the spec. Each one becomes a short ADR in `docs/adr/` during M1. The ones marked † deviate from the kickoff.

| Topic | Decision |
| --- | --- |
| Language and framework | **Java 25 + Spring Boot 4.1.2**, already fixed in `pom.xml`. † The kickoff said "Java 21+". |
| LLM | Claude API through Spring AI |
| Stockfish | † **Binary inside the app image**, driven over UCI as a local process. The kickoff put it in its own container, which would need a TCP bridge because UCI speaks stdin/stdout. Testcontainers uses the same image. |
| Chess rules | chesslib (or an equivalent JVM library) for legality, SAN parsing and FEN |
| Database | PostgreSQL + pgvector |
| Embeddings | † **Local ONNX embeddings** (Spring AI transformers, a MiniLM-class model). Claude has no embeddings API, and this avoids a second API key. |
| Chat memory | Spring AI chat memory persisted in PostgreSQL |
| MCP | Spring AI MCP server, **streamable HTTP**, inside the same app |
| Evals | † **Maven profile** `./mvnw verify -Pevals` in the single existing module. The kickoff's `-pl evals` assumed a multi-module build. Cases and results live in `evals/`. |
| CI | † Build + tests on every push and PR. Reduced evals only on push to `main` and through `workflow_dispatch`, not on every push. |
| Interface | Swagger UI plus a minimal static page |
| Infra | Docker Compose: app + PostgreSQL/pgvector |
| Specs location | † `docs/specs/` (this file is `docs/specs/mvp.md`). The CLAUDE.md seed should be updated to match, including the eval command. |

---

## 8. Risks

| Risk | Mitigation |
| --- | --- |
| **Spring AI is not yet validated with Spring Boot 4.1.2 and Java 25** (ChatClient + Anthropic, tool calling, pgvector, transformers embeddings, MCP server starter) | **Spike at the start of M2** that runs each of these end to end. If something is incompatible, pin the latest compatible versions or replace that piece, and record the outcome in an ADR. |
| Analyzing hundreds of games takes time | Configurable depth, async jobs, FEN cache |
| Claude API cost when running evals often | Reduced set in CI, full set by hand, tokens and cost logged per run |
| Lichess and Chess.com rate limits | Sequential requests, backoff on `429`, incremental import |
| Local embeddings are lower quality than hosted ones | Small, structured RAG documents; exact queries for numeric questions |
| Scope creep | Anything not in this spec goes to the backlog |
| The verifier only covers concrete claims | Documented as a known limitation in the README |

---

## 9. Open decisions

- [ ] **Public demo strategy:** video/GIF only, a precomputed read-only demo instance, or live chat with a rate limit. This covers public deployment, which stays optional and comes after M4.
- [ ] **Final project and repo name.**
- [ ] Exact Spring AI version (depends on the M2 spike).
