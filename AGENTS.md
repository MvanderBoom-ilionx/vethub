# AGENTS.md

Guidance for AI agents working in this repository. Every line answers: "Would an agent likely miss this?"

---

## Repo layout

```
vethub/
├── server/   Spring Boot 4 / Java 25 backend (Gradle)
├── client/   SvelteKit 2 / TypeScript frontend (Bun)
└── scripts/  Shell utilities (openapi-sync.sh, common.sh)
```

No workspace-level build. Each sub-project is independent; run commands from the correct directory.

---

## Backend (`server/`)

### Commands

```bash
# Run (dev profile active by default via mise.toml)
./gradlew bootRun

# All tests
./gradlew test

# Single test class
./gradlew test --tests "dev.ilionx.workshop.api.owner.service.OwnerServiceTest"

# Single test method
./gradlew test --tests "dev.ilionx.workshop.api.owner.service.OwnerServiceTest.shouldReturnOwnerWhenValidIdExists"

# Format + lint + static analysis (runs automatically before compile)
./gradlew spotlessApply check

# Format only
./gradlew spotlessApply

# Check without fixing
./gradlew spotlessCheck checkstyle pmd spotbugs
```

### Critical build quirk — `spotlessApply` runs on every compile

`JavaCompile` tasks `dependsOn("spotlessApply")`. **Every `./gradlew build` or `./gradlew test` auto-formats source files before compiling.** Do not be surprised when files are modified mid-build.

### Java compilation is strict

Compiler flags include `-Werror` — all warnings are errors. `-Xlint:all` is enabled except `-serial`, `-processing`, and `-this-escape`. Any new warning introduced by new code will fail the build.

### Database migrations

Liquibase changesets live in `server/src/main/resources/db/changelog/changesets/`. New migrations must be added there as XML files and registered in the master changelog. Two context tags control which changesets run:

- `prd` — schema DDL only
- `tst` — schema DDL plus seed data (used by `dev` and `test` profiles)

### Profiles and credentials

| Profile | Liquibase context | Credentials | When used |
|---------|------------------|-------------|-----------|
| `dev` (default) | `tst` (schema + seed data) | `user` / `password` | `bootRun` |
| `test` | `tst` | `sa` / (empty) | integration tests |
| (none / prd) | `prd` (schema only) | — | production jar |

Active profile is set via `SPRING_PROFILES_ACTIVE` in `mise.toml` (defaults to `dev`).

### Test context — important facts

- Integration tests boot a **real Spring context** on a random port with H2 `testdb`.
- `application-test.yml` sets `server.servlet.context-path: ""` (empty). In production/dev the context path is `/api`. MockMvc requests in tests do **not** need the `/api` prefix.
- `@BeforeEach` / `@AfterEach` in `IntegrationTest` cleans the database but **preserves seed data**: vets with ID ≤ 6, pet types with ID ≤ 6, specialties with ID ≤ 3. Tests that rely on seed data can use these IDs directly (e.g. `petTypeRepository.findById(1)`).
- Unit tests (`extends UnitTest`) use no Spring context; create mocks manually with `mock(...)` in `@BeforeEach` and inject via constructor — no `@Mock` / `@InjectMocks`.
- Test method naming follows the `should<Behaviour>When<Condition>` pattern (e.g. `shouldReturnOwnerWhenValidIdExists`).

### Package structure

Feature-first under `dev.ilionx.workshop.api.{owner,pet,vet,visit}`. Each domain contains `controller/`, `service/`, `repository/`, and `model/` (with `request/`, `response/`, `mapper/`, `validator/` sub-packages). Cross-cutting config lives in `common/`.

### MapStruct mappers

Declared as `abstract class`, not `interface`. Generated implementations are Spring beans via `SharedMapperConfig` (from `jframe` library). Mappers are only used in controllers — services work with entities directly.

### DTO conventions

Request and response DTOs are flat classes. Annotate all fields with `@Schema` (SpringDoc) so they appear correctly in the generated OpenAPI spec. Use Lombok (`@Getter`, `@Setter`, `@Builder`, etc.) to reduce boilerplate. Do **not** add Jakarta Bean Validation annotations (`@NotBlank`, `@Valid`) — validation is handled by explicit validator classes where needed.

### Validation

Owner domain uses a custom `OwnerValidator` (called explicitly in the controller). Pet, vet, visit, and specialty domains have **no validator** — this is an intentional workshop gap. Do not add `@Valid` / Bean Validation annotations without checking whether a custom validator approach is expected.

### Error handling

All "not found" errors throw `DataNotFoundException(ApiErrorCode.XYZ)` from the `jframe` library. The library serialises these to `404` responses automatically. Error codes live in `common/exception/ApiErrorCode.java`.

### URL constants

All paths are defined in `api/Paths.java`. Always import from there — never hard-code path strings.

### API is served at `/api/v1/...` in dev, but at `/v1/...` in tests

The servlet context path `/api` is stripped in the test profile. Use `Paths.*` constants in tests (they do not include `/api`).

---

## Frontend (`client/`)

### Commands

```bash
# Install dependencies
bun install

# Dev server (requires backend running on :8080)
bun run dev

# Type-check
bun run check

# Build
bun run build

# Regenerate TypeScript types from running backend
bun run sync:api        # starts backend, fetches spec, generates types, stops backend

# Just regenerate types (backend must already be running)
bun run generate:api    # openapi-typescript ../server/openapi.json -o src/lib/types/api.d.ts

# Download spec only (backend must be running)
bun run download:api
```

### Generated file — never edit manually

`src/lib/types/api.d.ts` is auto-generated by `openapi-typescript`. It is committed to the repo. After changing the backend API, run `bun run sync:api` and commit the updated file.

### Type import convention

Import types from `$lib/api/models` (friendly re-exports), not directly from `$lib/types/api`. The `models.ts` file is the canonical import point for all API types.

### Frontend domain controllers

Each domain has a thin `*Controller.ts` file (e.g. `OwnerController.ts`, `PetController.ts`) under `$lib/api/{domain}/`. These wrap `openapi-fetch` calls and normalise the `{ data, error }` return into thrown errors. When adding a new domain, follow the same naming pattern and place the file in the matching sub-directory.

### HTTP client

`openapi-fetch` is used via a singleton at `$lib/api/client.ts`. It is typed against `paths` from `api.d.ts` — passing a URL that does not exist in the spec is a compile error. Path parameters must be passed as `params: { path: { id } }`, not interpolated into the URL string.

### Backend base URL

Defaults to `http://localhost:8080/api`. Override via `VITE_SERVER_BASE_URL` env var. Dev credentials default to `user` / `password`; override via `VITE_API_USERNAME` / `VITE_API_PASSWORD`.

---

## OpenAPI sync workflow

When backend API changes, the update flow is:

1. `cd server && ./gradlew bootRun` (or use `bun run sync:api` which handles this automatically)
2. `cd client && bun run generate:api`
3. Commit the updated `server/openapi.json` and `client/src/lib/types/api.d.ts`

After regenerating, run `bun run check` to surface any frontend breakage caused by the backend change.

---

## Tooling

- **mise** manages tool versions: Java 25 (Temurin), Bun 1.3.0, Node 22.20. Run `mise install` if tools are missing.
- `SPRING_PROFILES_ACTIVE=dev` is set in `mise.toml` and picked up automatically when using mise.
- Quality config files live in `server/src/quality/config/` (Checkstyle, PMD, SpotBugs, Spotless). Do not modify them without understanding the ruleset.
