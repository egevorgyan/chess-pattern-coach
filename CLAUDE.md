# CLAUDE.md — Chess Pattern Coach

## Project
Java backend that imports a player's games from Lichess and Chess.com, analyzes
every move with Stockfish, detects recurring mistakes, and explains them with an
LLM whose concrete claims (moves, evaluations, classes) are verified against the
engine before reaching the user. It is a public portfolio project aimed at tech
leads at Java companies: code quality, tests, docs and measured eval metrics
matter as much as features. Scope and acceptance criteria: `docs/specs/mvp.md`.

## Stack
- Java 25, Spring Boot 4.1.2 (Web MVC), Maven 3.10 via wrapper, single module
- Tests: JUnit 5 (spring-boot-starter-webmvc-test)
- Planned (see spec §7): Spring AI + Claude API, PostgreSQL + pgvector, local
  ONNX embeddings, Stockfish binary inside the app image (UCI, local process),
  chesslib, Spring AI MCP server (streamable HTTP), Testcontainers (real
  Postgres and Stockfish; no live calls to Lichess, Chess.com or Claude in the
  default suite)
- CI: GitHub Actions · Local infra: Docker Compose
- Base package: `com.egevorgyan.chesspatterncoach`

## Commands
Use `mvnw.cmd` instead of `./mvnw` in Windows cmd/PowerShell.
- Build + all tests: `./mvnw verify`
- Run the app (port 8080): `./mvnw spring-boot:run`
- Single test class: `./mvnw test -Dtest=ClassName`
- Not yet available: `docker compose up` (M1), `./mvnw verify -Pevals` (M2)

## Docs layout
- `docs/kickoff.md` — original brief and rationale (historical; the spec wins on conflict)
- `docs/specs/` — specs; `mvp.md` holds user stories US-01…US-13 and milestones M1–M4
- `docs/architecture.md` — modules, boundaries, data flow (to be created)
- `docs/adr/` — ADRs as `NNNN-title.md`, e.g. `0001-stockfish-in-app-image.md`
- `docs/plan.md` — task list (to be created); global ids `T-01`, `T-02`…,
  each task with its milestone, stories and status

## Workflow
1. Spec → design (architecture.md / ADR) → plan (plan.md) → one task at a time.
2. Before any task, read the spec stories it implements and the relevant ADRs.
3. Plan before editing: state the approach and files to touch, then implement.
4. Work only on the current milestone; anything else goes to the backlog in plan.md.
5. A new technical decision, or a change to one, needs an ADR before the code.

## Conventions
- Everything in English: code, comments, docs, commits, PRs, README.
- No over-engineering: the simplest design that meets the spec.
- GitHub Flow: `main` is always green; branch `feat/t-01-<slug>`
  (lowercase task id); one PR per task; squash merge.
- Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`).
- Keep `Co-Authored-By:` trailers in commits and in squash-merge messages.

## Rules
- No LLM-stated move, evaluation or class reaches the user unverified (spec US-07).
- Every endpoint gets an integration test.
- Prompt changes require an eval run (from M2); do not merge if accuracy drops.
- Keep the README results table in sync with the latest full eval run.
- Never commit secrets; `ANTHROPIC_API_KEY` goes in `.env` (see `.env.example`).

## Definition of done
- `./mvnw verify` green locally and in CI.
- Acceptance criteria of the targeted stories covered by tests.
- Docs updated: spec/ADR/architecture if behavior or design changed; task
  status in `docs/plan.md`.
- PR description covers **What**, **Why** (story/ADR links) and **How verified**.
