# VetHub

VetHub is a veterinary clinic management application that allows clinic staff to manage owners, pets, vets, and visits. It is built as a decoupled monorepo with an independent backend and frontend that communicate over a typed HTTP REST contract. **This project exists for lab and workshop purposes only — it is not intended for production use.**

---

## Tech stack

### Server (`server/`)

| Concern | Technology |
|---------|-----------|
| Language | Java 25 (virtual threads enabled) |
| Framework | Spring Boot 4 (Web MVC, Data JPA, Security, Actuator) |
| Build | Gradle with Kotlin DSL |
| Database | H2 in-memory, Hibernate/JPA, HikariCP |
| Schema migrations | Liquibase |
| Object mapping | MapStruct 1.6.3 (compile-time) |
| Boilerplate reduction | Lombok |
| API documentation | SpringDoc OpenAPI 3 |
| Code quality | Spotless, Checkstyle, PMD, SpotBugs |

### Client (`client/`)

| Concern | Technology |
|---------|-----------|
| Language | TypeScript 5 |
| Framework | Svelte 5 + SvelteKit 2 |
| Runtime / build | Bun + Vite 7 |
| Styling | Tailwind CSS 4, shadcn-svelte, bits-ui |
| HTTP client | openapi-fetch (typed against generated spec) |
| Type generation | openapi-typescript |

---

## Repository layout

```
vethub/
├── server/   Spring Boot 4 / Java 25 backend (Gradle)
├── client/   SvelteKit 2 / TypeScript frontend (Bun)
└── scripts/  Shell utilities (openapi-sync.sh, common.sh)
```

Neither sub-project depends on the other at build time. They are connected only by an OpenAPI contract at runtime.

---

## Getting started

### 1. Fork the repository

1. Open the upstream repository at [https://github.com/Ilionx-AI/vethub](https://github.com/Ilionx-AI/vethub).
2. Click **Fork** (top-right) and follow the prompts to create a fork under your own GitHub account.

### 2. Clone your fork

Replace `<your-username>` with your GitHub username:

```bash
git clone https://github.com/<your-username>/vethub.git
cd vethub
```

Optionally, add the upstream remote so you can pull future changes:

```bash
git remote add upstream https://github.com/Ilionx-AI/vethub.git
```

### 3. Open in a dev container

The repository ships with a pre-configured dev container (`.devcontainer/`) that installs all required tooling automatically.

**Prerequisites:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) and the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) for VS Code.

1. Open the cloned folder in VS Code.
2. When prompted *"Reopen in Container"*, click it. Alternatively, open the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`) and run **Dev Containers: Reopen in Container**.
3. VS Code builds the container image and runs `mise install` automatically. Wait for the setup to complete before running any commands. The following tools are installed by mise (versions pinned in `mise.toml`):

   | Tool | Version | Purpose |
   |------|---------|---------|
   | Java (Temurin) | 25 | Backend runtime and compiler |
   | Bun | 1.3.0 | Frontend runtime, package manager, and test runner |
   | Node.js | 22.20.0 | Required by some frontend tooling |
   | OpenCode | latest | AI coding assistant (CLI) |

Ports `8080` (Spring Boot API) and `5173` (Vite dev server) are forwarded to your host automatically.

---

## Quick start

### Prerequisites

Tool versions are managed by [mise](https://mise.jdx.dev/). Run `mise install` from the repo root to install all required tools (Java 25, Bun 1.3.0, Node 22.20.0, OpenCode). When using the dev container this step runs automatically.

### Backend

Open a dedicated terminal and keep it running for the duration of your session:

```bash
cd server
./gradlew bootRun
```

Once started, verify the backend is healthy in a second terminal:

```bash
curl http://localhost:8080/api/actuator/health
# Expected: {"status":"UP"}
```

The API is available at `http://localhost:8080/api`. Default credentials: `user` / `password`.

### Frontend

With the backend running, open a second terminal for the frontend:

```bash
cd client
bun install
bun run dev
```

Verify the dev server is up in a third terminal:

```bash
curl http://localhost:5173
# Expected: HTML response from the SvelteKit dev server
```

The dev server starts at `http://localhost:5173` and requires the backend to be running on port 8080.

---

## Further reading

- [ARCHITECTURE.md](ARCHITECTURE.md) — detailed description of the backend and frontend structure, layer responsibilities, and the OpenAPI sync workflow.
- [AGENTS.md](AGENTS.md) — guidance for AI agents working in this repository.
