# return-shipment-tracker

A small backend service that tracks returned items as they move through a shipment lifecycle. It accepts grading events from a queue, keeps a status for each return, and records an audit trail of every status change.

This is a personal learning project, built to practice Java backend design with Spring Boot, event-driven messaging on AWS (SNS and SQS, run locally with LocalStack), and a small MCP server. It is not affiliated with any employer, and it does not use any employer's code or data.

## Status

Built in phases. Each phase is pushed when it works and has tests.

- [ ] **Phase 1:** core REST API, status lifecycle, trace events, tests
- [ ] **Phase 2:** SNS to SQS event flow with idempotent consumer, retries and a dead-letter queue
- [ ] **Phase 3:** MCP server (TypeScript) with tools to query shipment status
- [ ] **Later:** scheduled Lambda for stuck shipments, CI, architecture diagram

Tick a box only when the phase is finished and tested.

## Architecture

```
mock grading publisher
        |
        v
   SNS topic  --->  SQS queue  --->  consumer (Spring Boot)  --->  PostgreSQL
                       |                     |
                       v                     +--> return_requests
                   dead-letter queue         +--> trace_events

REST API (Spring Boot)  --->  PostgreSQL
MCP server (TypeScript) --->  REST API
```

## Tech

Java 17, Spring Boot, Spring Data JPA, PostgreSQL, Docker Compose, JUnit 5, Mockito, LocalStack (SNS and SQS), TypeScript with Zod for the MCP server.

## Status lifecycle

`INITIATED` to `PLANNED` to `SHIPPED` to `DELIVERED`. A request can move to `CANCELLED` or `ERROR` from any non-final state. Any other transition is rejected with `409` and records nothing.

Every successful status change adds one trace event (status, timestamp, message). Trace events are never edited or deleted.

## API

| Method | Path | Behavior |
|---|---|---|
| POST | `/returns` | Create a return request. `201` on success, `400` on invalid input. |
| GET | `/returns/{id}` | Current status and ordered trace events. `404` if unknown. |
| PATCH | `/returns/{id}/status` | Move to a new status. `409` on an illegal transition. |
| GET | `/returns?customerOrderId=...` | Find a request by customer order id. |
| GET | `/returns?stuck=true` | Requests that are not final and have not changed within the threshold. |

## Event handling

- A grading event creates exactly one return request.
- The same `eventId` delivered twice is acknowledged and ignored.
- A malformed message is rejected and logged, and is not retried.
- If processing fails because of an internal or dependency error, the message is retried and then moves to the dead-letter queue.
- If writing a trace event fails, the main flow still completes and the failure is logged.

## Run locally

> Fill this in as you build it, and test every command on a clean machine before you push.

```
# 1. Start Postgres and LocalStack
docker compose up -d

# 2. Run the service
./mvnw spring-boot:run

# 3. Run the tests
./mvnw test
```

## Design decisions

> Write these in your own words as you build. Short and specific beats long and generic.

**Why an SQS queue between SNS and the consumer?**
TODO

**Why does the consumer need to be idempotent?**
TODO

**What happens if the consumer crashes halfway through a message?**
TODO

**Why must trace failures not block the main flow?**
TODO

**What would change if this ran at much higher volume?**
TODO

## What I would do next

- Move the trace store to DynamoDB and compare it with the Postgres version.
- Run the flow as a Step Function with a scheduled Lambda for stuck shipments.
- Add CI with GitHub Actions and an architecture diagram.
