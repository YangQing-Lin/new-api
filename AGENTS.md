# Repository Guidelines

## Project Structure & Module Organization
- `main.go` boots the Go API; domain logic stays in `service/`, HTTP handlers in `controller/`, and request routing in `router/` with shared middleware in `middleware/`.
- `common/`, `constant/`, `dto/`, and `types/` hold cross-cutting utilities, config constants, and request/response contracts; prefer extending these packages instead of duplicating structs.
- `web/` contains the React administration UI (Vite + Bun). UI state lives under `web/src`, static assets under `web/public`, and brand resources under `docs/images`.
- Deployment assets live at the repo root (`Dockerfile`, `docker-compose.yml`, `new-api.service`); keep environment-specific overrides out of version control.

## Build, Test, and Development Commands
- `go run main.go` starts the API against the local configuration; use it for quick backend iteration.
- `make build-frontend` installs Bun deps and builds the Vite bundle with the release version from `VERSION`, while `make start-backend` boots the Go server afterward.
- `bun --cwd web run dev` serves the React UI with hot reload, proxying API calls to `localhost:3000`.
- `docker compose up --build` launches the full stack using the production images; mount persistent volumes for SQLite data.

## Coding Style & Naming Conventions
- Run `go fmt ./...` and `goimports` before sending patches; follow Go idioms: package names lower_snake, exported identifiers PascalCase, private members camelCase.
- Frontend code follows Prettier (`bun run lint:fix`) and ESLint (`bun run eslint:fix`); keep components PascalCase, hooks `useCamelCase`, and reuse i18n keys already under `web/src/locales`.

## Testing Guidelines
- Add Go tests in `*_test.go` files next to the code and run `go test ./...`; favor table-driven cases for controllers and services.
- Exercise frontend utilities with Vitest once added; mirror filenames as `*.test.ts(x)` in `web/src`.
- Before merging, run lint and test commands for both stacks; attach sample payloads for new endpoints when opening a PR.

## Commit & Pull Request Guidelines
- Follow the Conventional Commits style used in history (`feat:`, `fix:`, `chore:`) and keep scopes narrow, e.g. `feat(controller): add minimax tts route`.
- Squash experimental commits locally; final messages should describe behavior, not implementation.
- Pull requests must outline the change, link to any tracked issue, list test commands run, and attach UI screenshots or API samples when the user experience shifts.

## Security & Configuration Tips
- Store secrets in environment files echoed by `setting/` readers; never commit `.env` material.
- Update `README.*` summaries and `docs/` callouts when adding new third-party integrations or required environment variables.

- Encoding: All code and documentation files must use **UTF-8 without BOM** encoding
