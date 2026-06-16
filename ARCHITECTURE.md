# Architecture

VetHub is a veterinary clinic management application. It is structured as a decoupled monorepo: an independent Spring Boot backend and an independent SvelteKit frontend that communicate over HTTP REST.

---

## Repo layout

```
vethub/
├── server/   Spring Boot 4 / Java 25 backend (Gradle)
├── client/   SvelteKit 2 / TypeScript frontend (Bun)
└── scripts/  Shell utilities for OpenAPI sync
```

Neither sub-project depends on the other at build time. They are connected only by an OpenAPI contract at runtime.

---

## Tech stack

### Backend (`server/`)

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

### Frontend (`client/`)

| Concern | Technology |
|---------|-----------|
| Language | TypeScript 5 |
| Framework | Svelte 5 + SvelteKit 2 |
| Build | Vite 7 |
| Styling | Tailwind CSS 4, shadcn-svelte, bits-ui |
| HTTP client | openapi-fetch (typed against generated spec) |
| Type generation | openapi-typescript |

### Tooling

- **mise** manages tool versions (Java 25, Bun 1.3.0, Node 22.20)
- **`scripts/openapi-sync.sh`** automates the contract update workflow

---

## Backend structure

### Package organisation

The backend uses **package-by-feature**, not package-by-layer. Each domain is a self-contained vertical slice:

```
dev.ilionx.workshop
├── Application.java
├── api/
│   ├── Paths.java              ← all URL constants (single source of truth)
│   ├── owner/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   └── model/
│   │       ├── Owner.java      ← JPA entity
│   │       ├── request/        ← inbound DTOs
│   │       ├── response/       ← outbound DTOs
│   │       ├── mapper/         ← MapStruct mapper
│   │       └── validator/      ← custom validator
│   ├── pet/
│   ├── vet/
│   └── visit/
└── common/                     ← security config, error codes, OpenAPI config
```

The domains are `owner`, `pet`, `vet`, and `visit`. Each follows the same internal structure.

### Layers and their responsibilities

**Controller** — HTTP only. Maps path variables and request bodies, calls the validator (owner domain), delegates to the service, maps the result through the mapper, and returns a `ResponseEntity`. No business logic.

**Service** — Business logic and transactions. Resolves related entities from their own repositories, constructs or mutates JPA entities directly, and persists via the repository. Services work with entities, not DTOs.

**Repository** — Spring Data JPA interfaces. Standard `JpaRepository` inheritance for CRUD; derived query methods (e.g. `findByLastName`) for domain-specific queries.

**Model** — JPA entities plus the DTOs and mapper scoped to that domain. Request and response DTOs are flat classes annotated with Lombok and `@Schema`. Mappers are MapStruct `abstract class` (not `interface`) using `SharedMapperConfig`, which registers the generated implementation as a Spring bean.

### Request flow (example: `POST /api/v1/owners/{ownerId}/pets`)

```
HTTP request
  → DispatcherServlet (Spring Security filter chain: CORS, Basic Auth)
  → PetController.createPet(@PathVariable ownerId, @RequestBody request)
  → PetService.create(ownerId, request)           @Transactional
      → ownerRepository.findById(ownerId)          throws 404 if missing
      → petTypeRepository.findById(request.typeId) throws 404 if missing
      → new Pet() populated from request fields
      → petRepository.save(pet)                    Hibernate INSERT
  → PetMapper.toResponse(pet)                      entity → DTO
  → ResponseEntity 201 Created + JSON body
```

### Error handling

All "not found" cases throw `DataNotFoundException(ApiErrorCode.XYZ)`. The `jframe` library intercepts this and serialises it to a `404` response. Error codes are centralised in `common/exception/ApiErrorCode.java`.

### Validation

The owner domain uses an explicit `OwnerValidator` component called in the controller before the service. Other domains (pet, vet, visit) have no validator — an intentional workshop gap. There are no Jakarta Bean Validation annotations (`@NotBlank`, `@Valid`) on any request DTOs.

### Database schema

Liquibase manages schema versioning via XML changesets in `server/src/main/resources/db/changelog/changesets/`. Two changesets, controlled by context tags:

- `prd` context — schema DDL only (7 `CREATE TABLE` statements)
- `tst` context — schema DDL plus seed data (owners, pets, vets, visits)

The `dev` and `test` profiles both apply the `tst` context. Production applies `prd` only.

---

## Frontend structure

```
client/src/
├── lib/
│   ├── api/
│   │   ├── client.ts           ← openapi-fetch singleton (Basic Auth, base URL)
│   │   ├── models.ts           ← re-exports all generated types with friendly names
│   │   ├── owner/OwnerController.ts
│   │   ├── pet/PetController.ts
│   │   ├── vet/VetController.ts
│   │   ├── visit/VisitController.ts
│   │   ├── pet-type/PetTypeController.ts
│   │   └── specialty/SpecialtyController.ts
│   ├── types/
│   │   └── api.d.ts            ← AUTO-GENERATED, never edit manually
│   ├── components/             ← Svelte UI components
│   └── config/constants.ts     ← base URL, credentials (env var overrides)
└── routes/                     ← SvelteKit file-based routing
```

### HTTP layer

A single `openapi-fetch` client instance is created in `client.ts`, typed against the `paths` interface from the generated `api.d.ts`. Every domain has a thin controller file that wraps client calls and normalises the `{ data, error }` return into thrown errors. Components call these controller functions directly.

### Type import convention

Always import from `$lib/api/models`, not from `$lib/types/api` directly. `models.ts` is the stable re-export layer over the generated file.

---

## How the layers connect

### At runtime

```
Browser
  → SvelteKit route
  → domain controller (e.g. OwnerController.ts)
  → openapi-fetch client  →  HTTP/JSON  →  Spring Boot (/api/v1/...)
                                             → controller → service → repository → H2
```

Authentication is HTTP Basic Auth. Credentials are hardcoded for dev (`user` / `password`) and overridable via environment variables.

### The OpenAPI contract

The backend exposes its spec at `/api/v1/public/docs` (SpringDoc). The frontend's type system is derived from this spec:

```
Spring controllers + @Schema annotations
  → SpringDoc generates OpenAPI JSON at runtime
  → scripts/openapi-sync.sh fetches spec → server/openapi.json
  → openapi-typescript generates client/src/lib/types/api.d.ts
  → openapi-fetch<paths> enforces the contract at compile time
```

A breaking change in the backend API (renamed field, removed endpoint) becomes a TypeScript compile error in the frontend after regenerating. Run `bun run check` to surface these.

### Update workflow when the backend API changes

```bash
# From client/
bun run sync:api   # starts backend, fetches spec, generates types, stops backend
bun run check      # verify no frontend breakage
# commit server/openapi.json and client/src/lib/types/api.d.ts
```

---

## Key conventions

- **All URL paths** are defined in `server/src/main/java/dev/ilionx/workshop/api/Paths.java`. Never hard-code a path string.
- **Mappers live in the model layer**, not the service layer. Services return entities; controllers map them to DTOs.
- **Tests mirror the main package structure** exactly. Unit tests extend `UnitTest` (Mockito only, no Spring). Integration tests extend `IntegrationTest` (full Spring context, MockMvc, real H2).
  - `UnitTest`: no Spring context. Instantiate the class under test directly in `@BeforeEach`, create dependencies with `mock(...)`, and inject them via constructor. Do **not** use `@Mock` or `@InjectMocks`.
  - `IntegrationTest`: boots a real Spring context on a random port with H2. Use MockMvc for HTTP assertions. `@BeforeEach`/`@AfterEach` cleans the DB but preserves seed data (pet types 1–6, vets 1–6, specialties 1–3).
- **Test method naming** follows the `should<Behaviour>When<Condition>` pattern (e.g. `shouldReturnOwnerWhenValidIdExists`).
- **Seed data IDs are stable**: pet types 1–6, vets 1–6, specialties 1–3. Integration tests can reference these directly; the cleanup hook preserves them.
- **Frontend domain controllers** follow the `*Controller.ts` naming pattern (e.g. `OwnerController.ts`) and live under `$lib/api/{domain}/`. When adding a new domain, create the controller file in the matching sub-directory.
- **The servlet context path `/api` is absent in tests**. `application-test.yml` sets it to empty. MockMvc requests use `Paths.*` constants directly (e.g. `/v1/owners`, not `/api/v1/owners`).
