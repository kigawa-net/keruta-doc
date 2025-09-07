# CLAUDE.md

日本語を使う
This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the keruta-doc repository, which serves as the centralized documentation hub for the keruta project ecosystem.
It contains design documents, operational guides, and technical specifications for multiple interconnected services.

## Architecture

The repository is organized into several key subprojects:

- **keruta**: Main API server documentation (Spring Boot/Kotlin-based system)
- **keruta-github**: GitHub integration tools and services
- **keruta-agent**: Kubernetes Job execution management CLI tool
- **keruta-admin**: Administrative panel and management interface
- **keruta-builder**: Build system and tooling
- **common**: Shared API specifications and OpenAPI documentation

## Key Integration Points

### OpenAPI Synchronization

The repository has automated OpenAPI spec synchronization via GitHub Actions (`.github/workflows/openapi-sync.yml`):

1. Pulls from `kigawa-net/keruta-api` repository
2. Generates OpenAPI documentation using `./gradlew openApiGenerate`
3. Generates TypeScript client for `kigawa-net/keruta-admin` using `npm run generate-client`
4. Automatically commits and pushes changes to both repositories

### Related Repositories

- `kigawa-net/keruta-api`: Main API server (Spring Boot/Kotlin, Gradle-based)
- `kigawa-net/keruta-admin`: Admin panel (Node.js/React-based)

## Common Commands

Since this is a documentation repository, there are no typical build/test/lint commands. The main automation occurs
through GitHub Actions for OpenAPI synchronization.

## Documentation Structure

The documentation follows a hierarchical structure:

- System-level documentation in `keruta/system/`
- API specifications in `common/apiSpec/`
- Component-specific documentation in respective subdirectories
- Each major component has its own README.md with specific guidance

## Working with OpenAPI Specs

The OpenAPI specification is located at `common/apiSpec/openapi.yaml` and is automatically generated from the keruta-api
server. Do not manually edit this file as it will be overwritten by the synchronization process.

## Integration Architecture

The keruta system consists of:

- **Kubernetes integration** for job execution and container orchestration
- **Git repository management** for source code handling
- **Task queue system** for asynchronous job processing
- **Agent-based architecture** for distributed task execution
- **Authentication via Keycloak** integration
- **Log streaming and monitoring** capabilities

When working with documentation updates, ensure consistency across related documents and maintain the established
organizational structure.
