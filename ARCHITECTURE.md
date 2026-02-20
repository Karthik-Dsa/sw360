[//]: # (Copyright Contributors to the Eclipse SW360 project, 2026.)
[//]: # (Part of the SW360 Portal Project.)
[//]: # ()
[//]: # (This program and the accompanying materials are made)
[//]: # (available under the terms of the Eclipse Public License 2.0)
[//]: # (which is available at https://www.eclipse.org/legal/epl-2.0/)
[//]: # (SPDX-License-Identifier: EPL-2.0)

# SW360 Architecture

This document describes the high-level architecture of the SW360 project — an SBOM and license compliance management platform for open source software.

## Table of Contents

1. [System Overview](#system-overview)
2. [Module Structure](#module-structure)
3. [Architecture Layers](#architecture-layers)
4. [Component Interactions](#component-interactions)
5. [Key Domain Entities](#key-domain-entities)
6. [Database Layer](#database-layer)
7. [REST API](#rest-api)
8. [Authentication & Authorization](#authentication--authorization)
9. [Background Services](#background-services)
10. [Configuration](#configuration)

---

## System Overview

SW360 is a backend server with a REST API that manages software components, projects, licenses, and vulnerability data. It stores data in CouchDB and exposes it through Apache Thrift-based internal services and a Spring Boot-based REST API.

```
  External Clients
       │
       ▼
┌─────────────────┐
│   REST API      │  Spring Boot 3.x  (rest/resource-server)
│  (Port 8080)    │  OAuth2 / Bearer token / Basic auth
└────────┬────────┘
         │  Thrift RPC over HTTP
         ▼
┌─────────────────┐
│ Backend Services│  Apache Thrift 0.20 services  (backend/)
│  (Port 8080)    │  One service per domain
└────────┬────────┘
         │  Cloudant SDK (HTTP)
         ▼
┌─────────────────┐
│    CouchDB      │  Document store
│  (Port 5984)    │  JSON documents
└─────────────────┘
```

---

## Module Structure

```
sw360/
├── backend/          # Apache Thrift service implementations
│   ├── attachments/  # File attachment management
│   ├── components/   # Component & Release service
│   ├── configurations/ # Runtime configuration service
│   ├── cvesearch/    # CVE search integration
│   ├── fossology/    # Fossology license scanner integration
│   ├── health/       # Health-check service
│   ├── licenses/     # License & Obligation service
│   ├── moderation/   # Moderation request workflow
│   ├── packages/     # Package (pURL) service
│   ├── projects/     # Project service
│   ├── schedule/     # Scheduled task service
│   ├── search/       # Cross-entity search service
│   ├── users/        # User management service
│   ├── vendors/      # Vendor service
│   └── vulnerabilities/ # Vulnerability service
│
├── libraries/        # Shared libraries
│   ├── datahandler/  # Database handlers, Thrift definitions, repositories
│   ├── commonIO/     # Common I/O utilities
│   ├── exporters/    # Data export (Excel, CSV)
│   ├── importers/    # Data import (SPDX, CycloneDX)
│   └── nouveau-handler/ # Full-text search (Lucene/Nouveau)
│
├── rest/             # REST API layer
│   ├── resource-server/    # REST endpoints (Spring Boot)
│   └── authorization-server/ # OAuth2 authorization server
│
├── clients/          # Java client SDK for the REST API
├── keycloak/         # Keycloak user storage provider & event listeners
├── config/           # CouchDB, Keycloak, SW360 configuration files
├── scripts/          # Build, test, migration, and utility scripts
└── third-party/      # Thrift compiler, licenses
```

---

## Architecture Layers

SW360 follows a strict layered architecture. Each layer may only call the layer directly below it.

```
┌──────────────────────────────────────────────────────────┐
│                    REST Controllers                       │
│  *Controller.java  –  handle HTTP requests & responses   │
└───────────────────────────┬──────────────────────────────┘
                            │ calls
┌───────────────────────────▼──────────────────────────────┐
│                    REST Services                          │
│  Sw360*Service.java  –  orchestrate Thrift calls          │
└───────────────────────────┬──────────────────────────────┘
                            │ Apache Thrift RPC
┌───────────────────────────▼──────────────────────────────┐
│                  Thrift Handlers                          │
│  *Handler.java  –  implement Thrift service interfaces    │
└───────────────────────────┬──────────────────────────────┘
                            │ calls
┌───────────────────────────▼──────────────────────────────┐
│                Database Handlers                          │
│  *DatabaseHandler.java  –  business logic, validation     │
└───────────────────────────┬──────────────────────────────┘
                            │ calls
┌───────────────────────────▼──────────────────────────────┐
│                  Repositories                             │
│  *Repository.java  –  CouchDB queries, views, indexes     │
└───────────────────────────┬──────────────────────────────┘
                            │ Cloudant SDK
┌───────────────────────────▼──────────────────────────────┐
│                    CouchDB                                │
│  Document store  –  JSON documents, design documents      │
└──────────────────────────────────────────────────────────┘
```

### Naming Conventions by Layer

| Layer | Pattern | Example |
|-------|---------|---------|
| REST Controller | `*Controller.java` | `ProjectController.java` |
| REST Service | `Sw360*Service.java` | `Sw360ProjectService.java` |
| Thrift Handler | `*Handler.java` | `ProjectHandler.java` |
| Database Handler | `*DatabaseHandler.java` | `ProjectDatabaseHandler.java` |
| Repository | `*Repository.java` | `ProjectRepository.java` |

---

## Component Interactions

### Creating a Component (example flow)

```
Client
  │  POST /api/components
  ▼
ComponentController.createComponent()
  │  calls Sw360ComponentService.createComponent()
  ▼
Sw360ComponentService
  │  calls ThriftClients.makeComponentClient()
  │  calls ComponentService.Iface.addComponent()
  ▼
ComponentHandler.addComponent()                   [backend/components]
  │  calls ComponentDatabaseHandler.addComponent()
  ▼
ComponentDatabaseHandler.addComponent()
  │  validates, sets metadata, calls repository
  ▼
ComponentRepository.add()                         [libraries/datahandler]
  │  Cloudant SDK HTTP call
  ▼
CouchDB (sw360db)
```

### Inter-Service Communication

Backend services communicate with each other using Apache Thrift over HTTP. The `ThriftClients` class (in `libraries/datahandler`) provides factory methods for each service client:

```java
ComponentService.Iface client = thriftClients.makeComponentClient();
ProjectService.Iface    client = thriftClients.makeProjectClient();
LicenseService.Iface    client = thriftClients.makeLicenseClient();
// ...
```

---

## Key Domain Entities

All entities are defined as Apache Thrift structs in `libraries/datahandler/src/main/thrift/`.

| Entity | Thrift File | Description |
|--------|-------------|-------------|
| `Component` | `components.thrift` | A software component (e.g., Apache Commons) |
| `Release` | `components.thrift` | A specific version of a component |
| `Project` | `projects.thrift` | A product or project using releases |
| `License` | `licenses.thrift` | An SPDX or custom license |
| `Obligation` | `licenses.thrift` | A license obligation/condition |
| `Vulnerability` | `vulnerabilities.thrift` | A CVE or security vulnerability |
| `Package` | `packages.thrift` | A distribution of a release with a pURL |
| `Attachment` | `attachments.thrift` | A file attached to a component/release/project |
| `User` | `users.thrift` | A SW360 user with roles |
| `Vendor` | `sw360.thrift` | A software vendor |
| `ModerationRequest` | `moderation.thrift` | A pending change awaiting moderation |

### Entity Relationships

```
Project
  └── linked Projects (tree structure)
  └── linked Releases (with relationship type)
        └── Component (parent)
        └── Vendor
        └── Attachments
        └── Licenses (main + other)
        └── Packages (via pURL)
        └── Vulnerabilities

License
  └── Obligations
```

### Clearing States

| Entity | State Enum | Values |
|--------|------------|--------|
| Release | `ClearingState` | NEW_CLEARING → UNDER_CLEARING → REPORT_AVAILABLE → APPROVED |
| Project | `ProjectClearingState` | OPEN → IN_PROGRESS → CLOSED |

---

## Database Layer

### CouchDB Databases

SW360 uses three CouchDB databases:

| Database | Property | Purpose |
|----------|----------|---------|
| `sw360db` | `couchdb.database` | All domain entities (components, projects, licenses, etc.) |
| `sw360users` | `couchdb.usersdb` | User documents |
| `sw360attachments` | `couchdb.attachments` | Binary file attachments |

### Repository Pattern

All repositories extend `DatabaseRepositoryCloudantClient<T>` and use CouchDB design documents (views) and Mango indexes for queries.

```java
public class ComponentRepository extends DatabaseRepositoryCloudantClient<Component> {
    // CouchDB map-reduce views for common queries
    // Mango indexes for filtering/sorting
}
```

### Query Patterns

Queries use one of two mechanisms:

1. **MapReduce Views** (design documents) — for aggregate queries and key-based lookups
2. **Mango Selectors** — for ad-hoc filtering using JSON query syntax

```java
// MapReduce view query
List<Component> results = queryView("byName", searchName);

// Mango selector query
Map<String, Object> selector = and(eq("type", "component"), eq("name", name));
List<Component> results = db.queryBySelector(selector, Component.class);
```

---

## REST API

### Overview

The REST API is implemented in `rest/resource-server/` using Spring Boot 3.x and Spring HATEOAS. It exposes hypermedia-driven (HAL) JSON responses.

### Endpoint Structure

Base path: `/api`

| Resource | Endpoint | Controller |
|----------|----------|------------|
| Projects | `/api/projects` | `ProjectController` |
| Components | `/api/components` | `ComponentController` |
| Releases | `/api/releases` | `ReleaseController` |
| Licenses | `/api/licenses` | `LicenseController` |
| Vulnerabilities | `/api/vulnerabilities` | `VulnerabilityController` |
| Packages | `/api/packages` | `PackageController` |
| Users | `/api/users` | `UserController` |
| Vendors | `/api/vendors` | `VendorController` |
| Obligations | `/api/obligations` | `ObligationController` |
| Attachments | `/api/attachments` | `AttachmentController` |
| Clearing Requests | `/api/clearingrequests` | `ClearingRequestController` |
| Moderation Requests | `/api/moderationrequests` | `ModerationRequestController` |
| Search | `/api/search` | `SearchController` |

### Response Format

All responses use HAL (Hypertext Application Language) with embedded resources and navigational links:

```json
{
  "_embedded": {
    "sw360:components": [ ... ]
  },
  "_links": {
    "self": { "href": "https://host/api/components" }
  },
  "page": {
    "size": 10,
    "totalElements": 42,
    "totalPages": 5,
    "number": 0
  }
}
```

### OpenAPI Documentation

Interactive API documentation (Swagger UI) is available at `/` of the resource server. The OpenAPI specification is auto-generated from Spring REST Docs and controller annotations.

---

## Authentication & Authorization

### Authentication Methods

SW360 supports three authentication methods:

| Method | Description |
|--------|-------------|
| **Keycloak JWT** | Primary method. OAuth2/OIDC tokens issued by Keycloak |
| **Bearer Token** | Per-user API tokens generated and stored in CouchDB |
| **Basic Auth** | Username/password (development only) |

### Authorization Model

#### Endpoint-Level Security

REST endpoints use Spring Security's `@PreAuthorize` annotation:

```java
@PreAuthorize("hasAuthority('WRITE')")   // requires WRITE authority
@PreAuthorize("hasAuthority('ADMIN')")   // requires ADMIN authority
```

#### Document-Level Permissions

Each document has visibility controls and moderator lists. Permission checks are performed by `DocumentPermissions` / `PermissionUtils`:

```java
makePermission(project, user).isActionAllowed(RequestedAction.WRITE)
```

#### User Roles (UserGroup enum)

Roles are hierarchical — higher roles include all permissions of lower roles:

```
ADMIN
  └── SW360_ADMIN
        ├── SECURITY_ADMIN
        ├── ECC_ADMIN
        └── CLEARING_ADMIN
              └── CLEARING_EXPERT
                    └── USER
```

#### Document Visibility Levels

| Level | Access |
|-------|--------|
| `PRIVATE` | Creator only |
| `ME_AND_MODERATORS` | Creator + moderators |
| `BUISNESSUNIT_AND_MODERATORS` | Same business unit + moderators (note: historical typo in enum name) |
| `EVERYONE` | All authenticated users |

---

## Background Services

### Moderation Workflow

When a user without write access edits a document, SW360 creates a `ModerationRequest` instead of applying the change directly. A moderator can then approve or reject the request.

```
User (no WRITE) → edits document
  │
  ▼
ModerationRequest created
  │
  ├── Moderator approves → change applied
  └── Moderator rejects → change discarded
```

### Scheduled Tasks

The `schedule` backend service manages recurring tasks such as:

- CVE search integration (vulnerability import)
- Fossology integration polling
- Database sanitation

### Full-Text Search

SW360 uses Apache Lucene via CouchDB Nouveau for full-text search. The `search` backend service exposes a unified search endpoint that queries across all entity types.

---

## Configuration

### Key Configuration Files

| File | Location | Purpose |
|------|----------|---------|
| `sw360.properties` | `/etc/sw360/sw360.properties` | Main SW360 configuration |
| `couchdb.properties` | `/etc/sw360/couchdb.properties` | CouchDB connection settings |
| `application.yml` | `rest/resource-server/src/main/resources/` | Spring Boot REST configuration |
| `keycloak.conf` | `/opt/keycloak/conf/` | Keycloak server configuration |

### Important sw360.properties Keys

```properties
# Backend Thrift server URL
backend.url=http://localhost:8080

# REST API token settings
rest.apitoken.write.generator.enable=true
rest.apitoken.read.validity.days=90
rest.apitoken.write.validity.days=30

# Access control
rest.write.access.usergroup=SW360_ADMIN
rest.admin.access.usergroup=SW360_ADMIN

# Feature flags
enable.flexible.project.release.relationship=true

# External integrations
fossology.url=http://fossology:8081
cvesearch.host=https://cve.circl.lu
```

### CouchDB Configuration

```properties
couchdb.url=http://localhost:5984
couchdb.user=admin
couchdb.password=password
couchdb.database=sw360db
couchdb.usersdb=sw360users
couchdb.attachments=sw360attachments
```

---

## Further Reading

- [README.md](README.md) — Project overview and quick start
- [README_DOCKER.md](README_DOCKER.md) — Docker-based deployment guide
- [CONTRIBUTING.md](CONTRIBUTING.md) — Contribution guidelines
- [REST API Documentation](rest/resource-server/src/docs/asciidoc/index.adoc) — AsciiDoc API reference
- [Eclipse SW360 Website](https://eclipse.dev/sw360/) — Official project website
