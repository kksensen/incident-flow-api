# IncidentFlow

IncidentFlow is a backend REST API for managing support tickets, incidents, assignments and Service Level Agreements (SLAs).

The project models a real incident-management workflow in which users can create tickets, analysts can manage and resolve them, and the system keeps track of priority, status, ownership, deadlines and status changes.

The main objective is to build a maintainable backend application with real business rules rather than a simple CRUD system.

---

## Overview

Support and operations teams need more than a database containing tickets.

An incident-management system must be able to answer questions such as:

- Who created the incident?
- Who is responsible for handling it?
- What is its current status?
- What is its priority?
- When should it be resolved?
- Has its status changed?
- Is the incident approaching or exceeding its SLA?
- Which incidents are currently open, critical or assigned to a specific analyst?

IncidentFlow is designed around those requirements.

---

## Core Features

IncidentFlow is designed to provide:

- Ticket creation and management
- Incident priority management
- Ticket status workflow
- User and analyst assignment
- SLA-based deadlines
- Status change history
- Comments and ticket interactions
- Pagination and sorting
- Filtering by status
- Filtering by priority
- Filtering by assigned user
- Request validation
- Global exception handling
- Authentication and authorization
- REST API documentation
- Automated testing
- Database migrations
- Containerized execution
- Continuous Integration

---

## Technology Stack

### Backend

- Java 21
- Spring Boot 4.1.1
- Spring Web
- Spring Data JPA
- Spring Security
- Bean Validation
- Maven

### Database

- PostgreSQL
- Flyway

### Testing

- JUnit 5
- Mockito

### Infrastructure

- Docker
- Docker Compose

### API Documentation

- Swagger
- OpenAPI

### Development & Automation

- Git
- GitHub
- GitHub Actions

---

## Architecture

IncidentFlow follows a layered backend architecture.

```text
Client
  │
  ▼
Controller
  │
  ▼
Service
  │
  ▼
Repository
  │
  ▼
PostgreSQL
```

### Controller Layer

Responsible for the HTTP interface of the application.

Controllers:

- receive requests;
- validate API input;
- delegate operations to the service layer;
- return appropriate HTTP responses.

Business rules should not live inside controllers.

### Service Layer

Contains the application's business logic.

Examples include:

- ticket creation rules;
- status transitions;
- assignment rules;
- SLA calculation;
- ticket closing and reopening;
- generation of status history.

This layer represents the core behaviour of IncidentFlow.

### Repository Layer

Responsible for persistence and database access.

Spring Data JPA provides the integration between the domain model and PostgreSQL.

### Database

PostgreSQL stores the persistent state of the application.

Flyway is responsible for versioning and applying database schema migrations.

---

## Domain Model

IncidentFlow is centered around four main entities.

### User

Represents a user of the system.

A user can interact with incidents according to their permissions and role.

Users can participate in a ticket primarily as:

- the user who created it;
- the user responsible for handling it.

### Ticket

Represents a support ticket or incident.

A ticket contains information such as:

```text
id
title
description
priority
status
createdAt
updatedAt
assignedTo
createdBy
deadline
```

The `Ticket` entity is the central resource of IncidentFlow.

### Comment

Represents an interaction associated with a ticket.

Comments allow information related to an incident to remain connected to the incident itself instead of being handled through external communication.

### StatusHistory

Represents the history of status changes performed on a ticket.

Whenever a ticket changes status, IncidentFlow records the transition so that the lifecycle of the incident can be traced later.

---

## Ticket Status

Tickets can use the following statuses:

```text
OPEN
IN_PROGRESS
WAITING
RESOLVED
CLOSED
```

These states represent the lifecycle of an incident from creation to completion.

---

## Ticket Priority

IncidentFlow supports the following priority levels:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

Priority is not merely descriptive.

It participates in business rules such as SLA calculation and ticket urgency.

---

## Business Rules

The application contains domain rules that make the system more than a generic CRUD API.

### SLA

Ticket deadlines depend on incident priority.

For example:

```text
Higher priority
      │
      ▼
Shorter SLA
```

A critical incident therefore receives a more restrictive resolution deadline than a lower-priority incident.

SLA calculation belongs to the business layer rather than the controller or persistence layer.

### Status History

Every ticket status change generates a corresponding history record.

Conceptually:

```text
OPEN
  │
  ▼
IN_PROGRESS
  │
  ▼
RESOLVED
  │
  ▼
CLOSED
```

Each transition can be preserved through `StatusHistory`.

This makes the lifecycle of an incident auditable.

### Closed Tickets

A closed ticket is considered to have completed its normal lifecycle.

Certain modifications should therefore no longer be accepted once the ticket reaches the `CLOSED` state.

These restrictions are enforced through business rules in the service layer.

### Assignment

Tickets may be assigned to a user responsible for handling the incident.

Assignment information can also be used when retrieving and filtering tickets.

---

## REST API

The API exposes ticket resources through HTTP.

### Create a Ticket

```http
POST /tickets
```

Creates a new ticket.

### List Tickets

```http
GET /tickets
```

Returns tickets available through the API.

Pagination can be applied:

```http
GET /tickets?page=0&size=20
```

### Find a Ticket

```http
GET /tickets/{id}
```

Returns a ticket identified by its ID.

### Update a Ticket

```http
PUT /tickets/{id}
```

Updates an existing ticket according to the application's validation and business rules.

### Delete a Ticket

```http
DELETE /tickets/{id}
```

Removes a ticket when the operation is permitted by the application's rules.

---

## Filtering

IncidentFlow supports ticket retrieval based on operational criteria.

### By Status

```http
GET /tickets?status=OPEN
```

### By Priority

```http
GET /tickets?priority=CRITICAL
```

Filtering by responsible user is also part of the API design.

Pagination and sorting can be combined with ticket searches so the API remains practical as the amount of stored data increases.

---

## DTOs

IncidentFlow separates its persistence entities from its external API representation.

Instead of exposing JPA entities directly, the API uses request and response objects such as:

```text
TicketRequest
TicketResponse
```

This separation helps prevent the persistence model from becoming tightly coupled to the public API contract.

The flow becomes:

```text
HTTP Request
     │
     ▼
Request DTO
     │
     ▼
Service / Domain
     │
     ▼
Entity
     │
     ▼
Database
```

Responses follow the inverse direction before being returned to the client.

---

## Validation

Incoming data is validated before being processed by the business layer.

Examples of expected validation include:

- mandatory ticket title;
- minimum description requirements;
- valid status values;
- valid request data.

Bean Validation is used to express and enforce these constraints.

Invalid client input should result in appropriate HTTP error responses rather than invalid domain state.

---

## Exception Handling

IncidentFlow uses centralized exception handling with Spring's `@ControllerAdvice`.

This keeps controllers focused on HTTP operations and prevents repetitive exception-handling code.

A typical API error response can follow a structure such as:

```json
{
  "timestamp": "2026-09-15T18:30:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Ticket not found",
  "path": "/tickets/123"
}
```

Centralized handling can cover cases such as:

- resource not found;
- invalid request data;
- invalid business operations;
- unauthorized access;
- forbidden operations.

---

## Security

IncidentFlow uses Spring Security for authentication and authorization.

The authorization model is based on application roles.

```text
ADMIN
ANALYST
USER
```

### USER

Represents a regular system user.

Typical responsibilities include creating and interacting with tickets within their permissions.

### ANALYST

Represents a user responsible for working with incidents.

Analysts can participate in ticket handling, assignment and incident resolution workflows according to the defined authorization rules.

### ADMIN

Represents administrative access to the system.

Administrative operations can be protected independently from regular user and analyst operations.

The exact authorization rules are enforced by the backend rather than relying on the client interface.

---

## Database Migrations

IncidentFlow uses Flyway to version database changes.

Instead of manually modifying production databases, schema changes are represented as migration files.

Conceptually:

```text
Application starts
      │
      ▼
Flyway checks migrations
      │
      ▼
Pending migrations are applied
      │
      ▼
Spring Boot application continues
```

This provides a reproducible database structure across environments.

---

## Testing Strategy

Automated tests focus primarily on business behaviour.

The most important test targets are the service-layer rules, including components responsible for:

- tickets;
- users;
- SLA calculations;
- status changes;
- invalid operations.

The testing stack uses:

```text
JUnit 5
Mockito
```

A typical unit-test structure follows:

```text
Arrange
  │
  ▼
Act
  │
  ▼
Assert
```

The objective is not merely to increase test count, but to verify rules whose failure would change application behaviour.

---

## API Documentation

IncidentFlow uses Swagger/OpenAPI to expose interactive API documentation.

The documentation should allow developers to understand and test the API without needing to inspect the source code first.

Through the API documentation, operations such as the following can be explored:

- Create ticket
- Retrieve tickets
- Update ticket information
- Change ticket status
- Add ticket interactions

---

## Docker

The application is designed to run in containers.

Docker is used to package the backend application, while Docker Compose coordinates the services required by the project.

Conceptually:

```text
Docker Compose
│
├── IncidentFlow API
│
└── PostgreSQL
```

This reduces environment differences between development and deployment.

---

## Continuous Integration

GitHub Actions is used for Continuous Integration.

The expected pipeline is:

```text
Push / Pull Request
        │
        ▼
      Build
        │
        ▼
      Tests
        │
        ▼
     Result
```

Automated builds and tests help identify broken changes before they are integrated into the main codebase.

---

## Project Structure

The application follows separation of responsibilities between its main layers.

A representative package organization is:

```text
src/
└── main/
    └── java/
        └── ...
            ├── controller/
            ├── service/
            ├── repository/
            ├── domain/
            ├── dto/
            ├── exception/
            └── config/
```

The exact structure may evolve with the implementation, but the architectural principle remains the same:

> HTTP concerns, business rules and persistence concerns should remain separated.

---

## Main Technical Decisions

### Layered Architecture

A layered architecture keeps HTTP handling, business logic and persistence responsibilities independent.

This makes the application easier to understand, test and evolve.

### DTOs Instead of Exposing Entities

JPA entities represent persistence concerns.

API contracts represent communication concerns.

Keeping them separate avoids unnecessary coupling between the database model and external clients.

### PostgreSQL

IncidentFlow deals with structured and relational domain data such as users, tickets, assignments and history records.

PostgreSQL provides the relational persistence required by this model.

### Flyway

Database structure is part of the application lifecycle and should therefore be versioned alongside the source code.

### Service-Layer Business Rules

Rules such as SLA calculation and ticket lifecycle restrictions belong to the service/domain layer instead of controllers.

### Monolithic Spring Boot Application

IncidentFlow is intentionally designed as a well-structured Spring Boot application rather than introducing distributed-system complexity before it is necessary.

Technologies such as message brokers, distributed caching and microservices are outside the initial scope.

The priority is a cohesive backend with clear responsibilities and reliable behaviour.

---

## Scope

The initial scope of IncidentFlow focuses on:

- REST API
- Business rules
- Persistence
- Security
- Validation
- Testing
- Documentation
- Containers
- CI

Technologies such as Kafka, RabbitMQ, Redis, Kubernetes and microservices are deliberately not required by the initial architecture.

They should only be introduced later if a concrete system requirement justifies the additional complexity.

---

## Technical Presentation

IncidentFlow can be presented technically through five main areas.

### 1. Domain

Explain the problem being modeled:

```text
Users
Tickets
Assignments
Priorities
Statuses
SLA
History
```

### 2. Request Flow

Explain how an API request moves through the application:

```text
Client
  ↓
Controller
  ↓
DTO
  ↓
Service
  ↓
Repository
  ↓
PostgreSQL
```

### 3. Business Logic

Demonstrate rules such as:

```text
Ticket priority
      ↓
SLA calculation
      ↓
Deadline
```

and:

```text
Status change
      ↓
Validation
      ↓
Ticket updated
      ↓
History generated
```

### 4. Quality

Explain how reliability is supported by:

- Bean Validation
- Exception handling
- JUnit
- Mockito
- Flyway

### 5. Delivery

Explain how the application can move from source code to a runnable system:

```text
GitHub
  ↓
GitHub Actions
  ↓
Build + Tests
  ↓
Docker
  ↓
Application + PostgreSQL
```

---

## Project Status

IncidentFlow is currently under development.

The project is being built incrementally, with each new capability being added around the same architectural principles:

- clear responsibilities;
- explicit business rules;
- testable behaviour;
- consistent API contracts;
- maintainable code.

---

## License

A license can be added according to the intended distribution and usage of the project.
