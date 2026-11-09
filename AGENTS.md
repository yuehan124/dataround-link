# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project overview

Dataround Link is an open-source, zero-code data integration platform:

- **Backend**: Spring Boot 3 / Java 17, MyBatis-Plus, Quartz, Hazelcast, PostgreSQL/H2.
- **Frontend**: React 18 + TypeScript + Vite + Ant Design 5.
- **Data engine**: Apache SeaTunnel. The backend generates SeaTunnel job JSON and submits it to a local SeaTunnel cluster over HTTP.
- **Connectors**: Pluggable table/file connectors discovered with Java `ServiceLoader`, packaged as separate JAR sets under `lib/connector/<version>`.

The packaged application serves the UI and API under the `/datalink` context path. The default HTTP port is `5600`; SeaTunnel runs on `8080`.

## Repository map

| Path | Purpose |
| --- | --- |
| `dataround-link-common` | Shared `Result`/`PageResult`, `BaseController`, JWT/cookie utilities, connector `Param`, constants, exceptions. |
| `dataround-link-connector` | Parent for connector plugins. `dataround-link-connector-base` defines the SPI and JDBC/file base classes. |
| `dataround-link-connector/dataround-link-connector-*` | One plugin per data source: MySQL, PostgreSQL, Oracle, SQL Server, TiDB, DM, Kingbase, Doris, ClickHouse, Hive, Kafka, file (local/FTP/SFTP). |
| `dataround-link-svc` | Main Spring Boot application: controllers, services, mappers, entities, Quartz scheduling, SeaTunnel config generators/client, schema SQL, packaging. |
| `dataround-link-web` | React frontend. Build output is written to backend static resources. |
| `docs` | Screenshots and release notes. |
| `pom.xml` | Root Maven reactor and dependency management. |
| `Dockerfile`, `docker-compose.yml`, `kubernetes.yml` | Deployment assets. |
| `dataround-link-svc/package.xml` | Assembly descriptor that lays out the distribution tarball (`bin/`, `conf/`, `lib/`, `seatunnel/`). |
| `dataround-link-svc/seatunnel-connectors-pom.xml` | Downloads SeaTunnel connector JARs during the build. |

## Build and run

### Prerequisites

- Java 17+
- Maven 3.8+
- Node.js 18+ and Yarn 1.x for frontend-only work

### Full build

```bash
mvn clean package -DskipTests
```

This produces `dataround-link-svc/target/dataround-link-<version>.tar.gz`. Note that this is a heavy build:

- The `dataround-link-svc` module builds the frontend with `frontend-maven-plugin` (downloads Node `v18.18.0` and Yarn `1.22.22`).
- It downloads SeaTunnel connector JARs via `seatunnel-connectors-pom.xml`.
- Network access is required the first time.

### Run the backend

The default profile is `prod`, which requires PostgreSQL. For local development use the H2 `test` profile:

```bash
# In an IDE: run io.dataround.link.LinkApplication with the test profile
# or after packaging:
./bin/start.sh --spring.profiles.active=test
```

Useful URLs:

- UI/API: `http://localhost:5600/datalink`
- Swagger UI: `http://localhost:5600/datalink/swagger-ui/index.html`
- SeaTunnel API: `http://localhost:8080`

`start.sh` starts the bundled SeaTunnel cluster first, then the Spring Boot app. The script sets `-Ddataround.link.homeDir=$BASE_PATH`, which `ConnectorFactory` uses to locate `lib/connector/<version>`.

### Frontend development

```bash
cd dataround-link-web
yarn install
yarn start    # http://localhost:5173/datalink, proxies /datalink/api to :5600
yarn build    # writes to ../dataround-link-svc/src/main/resources/static
yarn lint
```

`yarn build` and `yarn lint` must pass for frontend changes. The backend build also runs the frontend build.

### Tests

```bash
# Service/unit tests use JUnit 5 and the H2 `test` profile
mvn -pl dataround-link-svc -am test -Dtest=SHA256UtilTest -Dsurefire.failIfNoSpecifiedTests=false

# All service-module tests (after dependencies are installed / available)
mvn -pl dataround-link-svc -am test
```

**Do not blindly run `mvn test` at the repository root.** Connector module tests are JUnit 4 live-integration tests that connect to real MySQL/PostgreSQL/Oracle/SQL Server/Hive/FTP/SFTP endpoints with hard-coded local connection details. Treat their failures as environment issues unless you are specifically changing that connector.

Because the frontend build and SeaTunnel connector download are bound to `generate-resources`, even a `test` invocation in `dataround-link-svc` may run those steps. If they are unrelated to your change, expect the first run to be slow and network-dependent.

## Architecture and request flow

1. `AuthInterceptor` validates the JWT cookie, refreshes it when needed, and stores the current `UserResponse` in the request.
2. `PermissionInterceptor` is registered for `/api/**`. It currently loads resource-based API permissions but leaves enforcement commented out (`TODO`). Frontend menus are still filtered by `resKey` permissions.
3. Controllers extend `BaseController` and use `getCurrentUserId()`, `getCurrentProjectId()`, and `getCurrentUser()`.
4. Most business tables are project-scoped. Queries and saves must include `project_id = getCurrentProjectId()` where applicable.
5. Services extend MyBatis-Plus `ServiceImpl`; mappers extend `BaseMapper`.
6. Job execution: a `Job` is turned into SeaTunnel JSON by `JobConfigService` and a `JobConfigGenerator`, submitted by `SeaTunnelRestClient`, and tracked as a `JobInstance`. Quartz (`SchedulerService`/`JobRunner`) handles scheduled execution.

## Connector plugin model

Connector classes implement:

- `TableConnector` via `AbstractTableConnector` / `JdbcConnector` for databases.
- `FileConnector` via `AbstractFileConnector` for local/FTP/SFTP.

Key implementation rules:

- Each plugin must have a `src/main/resources/META-INF/services/io.dataround.link.connector.Connector` file listing its implementation class(es).
- JDBC connectors should extend `JdbcConnector` (or another JDBC connector, e.g. MySQL CDC extends MySQL) and implement `getName()` plus database-specific catalog/schema handling.
- `AbstractTableConnector` switches the thread context classloader and closes the connector after metadata calls. Do not bypass that lifecycle when adding metadata methods.
- Connector metadata is stored in the `connector` and `connector_version` tables. Runtime display names must match `ConnectorNameConstants` and `ConnectionFactory`.

### Adding a new connector

A complete connector addition touches all of the following:

1. New Maven module under `dataround-link-connector`, registered in `dataround-link-connector/pom.xml`.
2. SPI file under `META-INF/services/`.
3. `ConnectorNameConstants` (if it is a new connector name).
4. `ConnectionFactory` and a `ConnectionVo` subtype if it needs custom connection fields.
5. Seed rows in **both** `postgresql-schema.sql` and `h2-schema.sql` (`connector`, and `connector_version` if versioned).
6. Packaging mappings in `dataround-link-svc/package.xml` so its JARs land in `lib/connector/<dir>`.
7. A `JobConfigGenerator` if SeaTunnel source/sink config differs from the generic JDBC/Kafka/file generators.
8. Frontend connection create form and API/type support if it has connector-specific fields.

## Backend conventions

- Package root: `io.dataround.link`.
- Use constructor or field injection consistent with surrounding code; controllers commonly use `@Autowired` fields.
- Controllers: `@RestController`, `@RequestMapping("/api/<resource>")`, endpoint style is typically `/list`, `/saveOrUpdate`, `/{id}`, `/delete/{id}`.
- API wrappers: `Result<T>` (`code`, `data`, `msg`) or `PageResult<T>` (`total`, `data`). Some list endpoints return MyBatis-Plus `Page<T>` directly (e.g. `JobController.list`); match the existing endpoint style you are editing.
- Errors: throw domain exceptions (`LinkException`, `JwtTokenException`) or `RuntimeException` consistent with the surrounding service; use `MessageUtils.getMessage(...)` for localized messages.
- Entities: MyBatis-Plus annotations, `IDType.AUTO`, snake_case DB columns map to camelCase fields by convention. JSON columns use `JsonTypeHandler`; note it behaves differently for PostgreSQL (`JSONB`) and H2 (`JSON`).
- Use `@TableField(updateStrategy = FieldStrategy.IGNORED)` when `null` intentionally clears a column (see `Job.startTime`/`endTime`).
- Logging: Lombok `@Slf4j`.
- New Java files should keep the existing copyright/license header style used by adjacent files.
- The `maven-checkstyle-plugin` is configured with `google_checks.xml` but not bound to a lifecycle phase; do not reformat unrelated code just to satisfy it.

## Frontend conventions

- Source layout: `src/api`, `src/pages`, `src/components`, `src/hooks`, `src/layout`, `src/router`, `src/store`, `src/utils`, `src/locales`.
- All HTTP calls go through `src/api/*.ts`, using `http` from `src/utils` and `baseAPI` (`/datalink/api`).
- Axios interceptors unwrap the backend `Result` and display `msg` messages automatically. A non-JSON response is treated as a login failure.
- State: `valtio` stores in `src/store/index.tsx`; permission helpers are `hasPermission`, `hasAnyPermission`, `hasAllPermissions`.
- Pages are lazy-loaded through `src/router/config.tsx`. Menu entries need `resKey` values that match backend resources; groups are hidden when all children are filtered out.
- UI: Ant Design 5, colocated `.less` files, icons from `@ant-design/icons`.
- Every user-visible string must be added to **both** `src/locales/en/translation.json` and `src/locales/zh/translation.json`, and accessed with `useTranslation()` / `t(...)`.
- Access the current backend via the Vite proxy during development; do not hard-code `localhost:5600` in frontend code.

## Database and schema

- Schema files:
  - `dataround-link-svc/src/main/resources/postgresql-schema.sql` (production, `JSONB`, `ON CONFLICT`, sequences)
  - `dataround-link-svc/src/main/resources/h2-schema.sql` (test/dev, `JSON`, `MERGE`, `NEXT VALUE FOR`)
- Schema initialization runs on every startup (`spring.sql.init.mode: always`). Keep all statements idempotent (`CREATE TABLE IF NOT EXISTS`, `ON CONFLICT DO NOTHING`, `MERGE`).
- **Keep both schema files in sync.** Any table, column, seed connector, or permission/resource change must be applied to both, respecting each dialect.
- IDs use sequences starting at `10000`.
- Seed users/resources are part of the schema; be careful not to reset or duplicate them.

## Internationalization

- Backend messages: `dataround-link-svc/src/main/resources/messages.properties` (English) and `messages_zh_CN.properties`.
- Use `MessageUtils.getMessage("key")`; the locale comes from `LocaleContextHolder` and the frontend's `Accept-Language` header.
- Frontend messages: `dataround-link-web/src/locales/en/translation.json` and `zh/translation.json`.
- Add keys to both languages in the same change.

## Packaging and deployment

- `package.xml` controls the distribution layout. Connector JARs are copied from module `target/` directories into `lib/connector/<name>`, SeaTunnel is copied into `seatunnel/`, scripts into `bin/`, configs into `conf/`.
- `seatunnel-connectors-pom.xml` controls which SeaTunnel connector JARs are downloaded into the distribution.
- The `Dockerfile` does **not** build from source. It expects `dataround-link-svc/target/dataround-link-*.tar.gz` to exist and extracts it.
- The vendored `dataround-link-svc/src/main/resources/seatunnel/` tree is third-party SeaTunnel distribution content. Avoid editing it unless the task is specifically about SeaTunnel packaging/configuration.
- `dataround-link-svc/src/main/resources/static/` is generated frontend output and is gitignored. Do not edit it by hand.

## Guardrails

- Preserve existing behavior and formatting outside the scope of the task.
- Do not commit build artifacts, `node_modules`, `target`, `data/`, logs, or generated static assets.
- Do not add secrets or real credentials to source; connector integration tests contain placeholders and should stay that way.
- When changing an API shape, update the frontend API module and affected pages in the same change.
- When adding a project-scoped feature, enforce `project_id` filtering on the backend; do not rely on frontend filtering.
- Prefer targeted Maven test invocations over the full reactor test suite because connector tests require live external systems.
