# Dataround Link v0.9.1 Release Notes

Dataround Link v0.9.1 is a feature and maintenance release that adds authentication and authorization management, dynamic permission-based menus, and additional database connectors.

## Highlights

- **Authentication and Authorization** - Added JWT-based login, session handling, users, roles, resources, and project membership management.
- **Dynamic Menus** - Menus and routes are now filtered according to the current user's resource permissions.
- **New Connectors** - Added Doris and ClickHouse connector modules, with DM and KingbaseES connector support.
- **Connector Version Management** - Connectors can define and select driver versions, and the JDBC driver class is resolved from the selected connector version.
- **JDBC Compatibility Improvements** - Improved MySQL 5.x/8.x driver handling, unified SQL Server driver usage, and fixed connector packaging.
- **Job Fixes** - Fixed periodic job validation, SeaTunnel job status handling, and removed an unsupported Kafka upsert property.

## Improvements

- Added user, role, resource, and project member management pages.
- Added permission-aware route and menu filtering.
- Added a my-account page with password update support.
- Improved the login page, logout flow, and session redirection behavior.
- Updated the UI color scheme and scrollbar styling.
- Added custom connector version support to connection configuration.
- Improved H2 schema compatibility.
- Improved connector class loading and JDBC driver initialization.
- Added agent-oriented development documentation in `AGENTS.md`.

## Fixes

- Fixed packaging that omitted `mssql-jdbc-*.jar`.
- Fixed periodic job validation on the frontend.
- Fixed SeaTunnel jobs that could remain in a failing state.
- Fixed logout redirect behavior.
- Fixed Kafka connector configuration by removing the unsupported upsert property.

## Supported Connectors

- JDBC Connectors: MySQL, PostgreSQL, Oracle, SQL Server, TiDB, DM, KingbaseES, Doris, ClickHouse
- CDC Connectors: MySQL-CDC, SQLServer-CDC
- Hive
- Kafka
- File Connectors: LocalFile, FTP, SFTP, S3

## Build from Source

Requirements:

- Java 17 or higher
- Maven 3.8 or higher

```bash
git clone https://github.com/yuehan124/dataround-link.git
cd dataround-link
mvn clean package -DskipTests
```

The deployment package will be generated at:

```text
dataround-link-svc/target/dataround-link-0.9.1.tar.gz
```

## Release Notes

- This release is source-only. No binary assets are attached to the GitHub release.
- A new Docker image is not published for v0.9.1. Use the existing `dataround/link:0.9.0` image or build from source.
- Database tables are created automatically on startup for both PostgreSQL and H2.

## Feedback and Contributions

If you encounter any issues or have suggestions, please contact us through:

1. GitHub Issues: https://github.com/yuehan124/dataround-link/issues
2. GitHub Discussions: https://github.com/yuehan124/dataround-link/discussions

We welcome all forms of contributions, including code submissions, documentation improvements, feature suggestions, and bug reports.

Thank you for using Dataround Link!
