# BookLore AI Agent Instructions

## Project Overview

BookLore is a self-hosted book library manager with Spring Boot backend (Java 25) and Angular 21 frontend. Port 6060 default. Uses MariaDB, embedded UI in Spring JAR.

## Code Style

**Backend (Java/Spring Boot)**
- Always use Lombok: `@Getter`, `@Setter`, `@Builder`, `@AllArgsConstructor`, `@NoArgsConstructor`
- MapStruct for entity-DTO mapping with `componentModel = "spring"`, mappers in `org.booklore.mapper`
- Packages use singular nouns: `model`, `service`, `controller` (not plural)
- Entities have `Entity` suffix; DTOs have no suffix (e.g., `BookEntity` → `Book`)

**Frontend (Angular)**
- All components are standalone (`standalone: true`)
- Use `inject()` for DI, not constructor injection (see [book-cover.service.ts](booklore-ui/src/app/shared/services/book-cover.service.ts))
- Services use `providedIn: 'root'` for singletons
- PrimeNG 21 components for UI

## Architecture

**Backend Structure**
```
booklore-api/src/main/java/org/booklore/
├── controller/          # REST endpoints @RestController, all /api/v1/**
├── model/
│   ├── entity/         # JPA entities with Hibernate 7.2
│   └── dto/            # Subdirs: request/, response/, kobo/, settings/, etc.
├── repository/         # Spring Data JPA repos
├── service/            # Business logic, organized by domain (metadata/, kobo/, book/)
└── mapper/             # MapStruct interfaces
```

**Frontend Structure**
```
booklore-ui/src/app/
├── features/           # Domain modules (book, bookdrop, dashboard, readers, settings, stats)
├── shared/             # Reusable components/services
└── core/               # App-wide services (security, config)
```

## Build & Test Commands

**Backend**
```bash
./gradlew build             # Build Spring Boot app
./gradlew bootRun           # Run locally (requires MariaDB)
```

**Frontend**
```bash
npm run build               # Production build
npm run dev                 # Development server
npm test                    # Vitest tests
npm run lint                # ESLint
```

**Docker**
```bash
docker compose -f example-docker/compose.yml up -d --build
```

## Key Conventions

**Filtering Pattern**
- Use JPA Specifications with static factory methods (see [MobileBookSpecification.java](booklore-api/src/main/java/org/booklore/mobile/specification/MobileBookSpecification.java))
- Combine specs with `combine()` method and `.and()` chaining
- Each filter returns `Specification<BookEntity>`

**DTO Organization**
- Request DTOs in `model/dto/request/` (e.g., `CreatePhysicalBookRequest`)
- Response DTOs in `model/dto/response/` (e.g., `BookDeletionResponse`)
- Integration-specific DTOs in subdirs: `kobo/`, `komga/`, `sidecar/`

**Service Patterns**
- Domain-specific packages: `service/book/`, `service/metadata/`, `service/kobo/`
- Background tasks use `@Async` annotation
- Virtual threads enabled globally in [application.yaml](booklore-api/src/main/resources/application.yaml)

**Frontend Filtering**
- Filter logic in `features/book/components/book-browser/filters/`
- Main config in [book-filter.config.ts](booklore-ui/src/app/features/book/components/book-browser/book-filter/book-filter.config.ts)
- Use extractor functions that return filter options from `Book` objects
- Access nested properties via optional chaining: `book.primaryFile?.fileSizeKb`

## Integration Points

**External Integrations**
- Kobo sync: `/kobo/**` with `KoboAuthFilter` and device tokens
- OPDS feeds: `/api/v1/opds/**` with Basic Auth
- KOReader plugin: `/api/koreader/**` with progress sync
- Komga API: `/komga/api/**` for client compatibility

**Metadata Providers**
- All parsers extend `BookParser` interface in `service/metadata/parser/`
- Providers: Google Books, Open Library, Hardcover, GoodReads, Audible, Amazon, others
- Format extractors in `service/metadata/extractor/` (PDF, EPUB, MOBI, FB2, audiobook)

## Security

**Authentication**
- Main API: JWT tokens via `JwtAuthenticationFilter`
- OPDS/Kobo: Basic Auth with separate filter chains (see [SecurityConfig.java](booklore-api/src/main/java/org/booklore/config/security/SecurityConfig.java))
- Streaming endpoints: Query-param JWT tokens

**Authorization**
- Use `@PreAuthorize("@securityUtil.canEditMetadata()")` for method security
- Custom `@CheckBookAccess` for book-level permissions
- Check patterns in [SecurityUtil.java](booklore-api/src/main/java/org/booklore/config/security/SecurityUtil.java)

## Database

- Flyway migrations in `src/main/resources/db/migration/`
- Use `V{number}__{Description}.sql` naming
- MariaDB 11.4+ required
- Virtual threads for DB operations

## Common Patterns

**Entity Access in DTOs**
- Book DTO has nested `primaryFile: BookFile` object
- File size: `book.primaryFile?.fileSizeKb` (NOT `book.fileSizeKb`)
- Metadata: `book.metadata?.authors`, `book.metadata?.tags`

**API Endpoints**
- All REST endpoints start with `/api/v1/`
- Integration endpoints: `/kobo/`, `/api/kobo/`, `/komga/`, `/api/koreader/`, `/api/v1/opds/`
- Use `@RequestMapping` at class level, method-specific annotations below

## Testing

**When to Add Tests**
- Add tests when changes introduce new business logic
- Add tests when fixing bugs to prevent regression
- Update existing tests when modifying tested functionality
- Backend: JUnit 5 tests in `src/test/java/`
- Frontend: Vitest tests colocated with components

## Code Quality Standards

**Code Review Requirements**
- No useless comments - code should be self-documenting
- No dead code - remove unused imports, methods, or classes
- PR is scoped - one feature/fix per PR, avoid unrelated changes
