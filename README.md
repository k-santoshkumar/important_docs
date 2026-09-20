Production Agent Backend — Complete Implementation Plan

Goal

Build a production-grade, ChatGPT-like agent backend with FastAPI, PostgreSQL, Redis, Amazon Bedrock, Pydantic AI, optional LangGraph, Temporal, S3, OpenSearch, observability, authentication, memory, RAG, streaming, reliability controls, AWS deployment, and production testing.

Target scale:

~1 million users

~10,000 concurrent/parallel requests

Stateless API layer

Durable background agent execution

1. Target Architecture

Clients
  |
API Gateway / ALB
  |
FastAPI (stateless)
  |--------- PostgreSQL (source of truth)
  |--------- Redis (cache/limits/coordination)
  |--------- S3 (files/artifacts)
  |
Temporal
  |
Agent Runtime
  |--------- Pydantic AI
  |--------- optional LangGraph
  |
  |---- LLM Gateway ---- Bedrock
  |
  |---- Tool Registry ---- Services/APIs

Files -> S3 -> Ingestion -> Chunking -> Embeddings -> OpenSearch -> Retriever

OpenTelemetry -> Logs / Metrics / Traces

Core rule: FastAPI should create and expose runs, but should not own long-running agent execution. Temporal owns durable execution.

Phase 1 — Domain & Data Modeling

Core entities

Tenant

id

name

plan

status

created_at

updated_at

User

id

tenant_id

email

name

status

created_at

updated_at

Conversation

id

tenant_id

user_id

title

status

created_at

updated_at

Message

id

conversation_id

run_id

role

content

sequence_number

metadata

created_at

Roles:
system, user, assistant, tool

Agent

id

name

description

version

status

configuration

created_at

updated_at

Agent Run

id

tenant_id

user_id

conversation_id

agent_id

agent_version

workflow_id

model_id

status

started_at

completed_at

error_code

error_message

metadata

Statuses:
queued, running, completed, failed, cancelled, timeout

LLM Call

id

run_id

provider

model_id

input_tokens

output_tokens

latency_ms

status

error

created_at

Tool

id

name

description

version

status

configuration

Tool Call

id

run_id

tool_name

tool_version

arguments

status

result_reference

started_at

completed_at

error

idempotency_key

Memory

id

tenant_id

user_id

type

key

value

source

confidence

status

created_at

updated_at

expires_at

File

id

tenant_id

user_id

conversation_id

s3_key

filename

content_type

size

status

created_at

Document

id

file_id

status

parser

chunk_count

embedding_model

created_at

updated_at

Usage

id

tenant_id

user_id

run_id

model_id

input_tokens

output_tokens

estimated_cost

created_at

Audit Event

id

tenant_id

user_id

event_type

resource_type

resource_id

metadata

created_at

Core relationships

Tenant -> Users -> Conversations -> Messages
                         |
                         -> Agent Runs -> LLM Calls
                                      -> Tool Calls

User -> Memories
Conversation -> Files -> Documents -> Vector index

Important distinction:

Conversation memory = messages

Execution state = workflow/run state

Long-term memory = memories

Phase 2 — PostgreSQL & Migrations

Technology:

PostgreSQL

SQLAlchemy 2.x

Alembic

UUID primary keys

JSONB for flexible metadata

Tables:
tenants, users, conversations, messages, agents, agent_runs, llm_calls, tools, tool_calls, memories, files, documents, usage, audit_events

Add:

PK/FK constraints

unique constraints

NOT NULL constraints

appropriate status checks

tenant isolation constraints

indexes based on query patterns

Important indexes:

users: tenant_id, email, (tenant_id, email)

conversations: user_id, tenant_id, (user_id, updated_at)

messages: conversation_id, (conversation_id, sequence_number), run_id

agent_runs: conversation_id, user_id, tenant_id, status, created_at

memories: user_id, tenant_id, (type, key)

usage: (tenant_id, created_at), (user_id, created_at)

Implementation:

Create local PostgreSQL.

Configure SQLAlchemy.

Create models.

Configure Alembic.

Create initial migration.

Apply migration.

Seed development data.

Add repository tests.

Add transaction tests.

Phase 3 — Backend Structure

Use:

app/
├── api/
│   ├── routes/
│   ├── dependencies.py
│   └── middleware/
├── domain/
│   ├── users/
│   ├── tenants/
│   ├── conversations/
│   ├── messages/
│   ├── agents/
│   ├── runs/
│   ├── tools/
│   ├── memories/
│   └── files/
├── services/
├── repositories/
├── infrastructure/
│   ├── database/
│   ├── redis/
│   ├── s3/
│   └── bedrock/
├── agents/
├── workflows/
├── auth/
├── observability/
└── config/

Layering:

API -> Services -> Domain -> Repositories -> Infrastructure

Avoid:

FastAPI route -> direct SQLAlchemy

Agent -> direct SQLAlchemy

Agent -> raw Bedrock calls

Phase 4 — FastAPI Foundation

Implement:

application factory

configuration

dependency injection

exception handling

request IDs

structured logging

health endpoint

readiness endpoint

API versioning

Initial endpoints:

GET /health
GET /ready

Use /v1 for the public API.

Phase 5 — Authentication & Multi-Tenancy

Implement:

authentication

current user resolution

current tenant resolution

authorization

roles/permissions

tenant isolation

Never trust client-provided tenant/user IDs without authorization checks.

Phase 6 — Conversation System

Endpoints:

POST   /v1/conversations
GET    /v1/conversations
GET    /v1/conversations/{id}
PATCH  /v1/conversations/{id}
DELETE /v1/conversations/{id}

POST   /v1/conversations/{id}/messages
GET    /v1/conversations/{id}/messages

Rules:

preserve message ordering

sequence messages

validate ownership

paginate

never load unlimited conversation history

support metadata

Phase 7 — Agent & Run System

Separate agent definitions from executions.

Agent
  name
  version
  configuration
  tools

Run
  agent_version
  model
  status
  timestamps
  execution metadata

Endpoints:

GET  /v1/agents
GET  /v1/agents/{id}
GET  /v1/runs/{id}
POST /v1/runs/{id}/cancel

Phase 8 — LLM Gateway & Bedrock

Architecture:

Agent -> LLM Gateway -> Bedrock Adapter -> Amazon Bedrock

Gateway responsibilities:

model selection

retries

timeouts

token accounting

tracing

logging

configuration

concurrency limits

fallback policy where appropriate

Do not scatter Bedrock calls throughout the codebase.

Phase 9 — Pydantic AI Agent Layer

Use Pydantic AI for:

typed agent dependencies

structured outputs

validation

tool definitions

agent logic

Keep agent logic independent of FastAPI.

Use LangGraph only where explicit graph/state semantics are useful.

Phase 10 — Tool System

Architecture:

Tool Registry
  -> Authorization
  -> Tool Execution
  -> Service Layer
  -> Infrastructure

Each tool should define:

name

description

input schema

output schema

permissions

timeout

retry policy

idempotency behavior

The LLM must not have unrestricted infrastructure access.

Phase 11 — Memory

Treat three things separately.

Conversation memory

Stored as messages.

Execution state

Managed through Temporal/workflow state.

Long-term memory

Stored in memories.

Pipeline:

Conversation
 -> Memory Extraction
 -> Candidate Memory
 -> Validation / Policy
 -> Persistent Memory

Do not store every statement as long-term memory.

At run start:

User -> Relevant memories -> Agent context

Apply relevance filtering and token budgets.

Phase 12 — Files

Architecture:

Client -> FastAPI -> S3

PostgreSQL stores metadata; S3 stores file bytes.

Implement:

upload

ownership

content-type validation

size limits

signed URLs

deletion

security scanning where required

Phase 13 — Document Ingestion & RAG

Pipeline:

File
 -> S3
 -> Ingestion Worker
 -> Parser
 -> Text Extraction
 -> Chunking
 -> Embeddings
 -> OpenSearch

Track:
queued, processing, completed, failed

Retriever abstraction:

class Retriever:
    async def search(...):
        ...

Agent uses a retriever tool rather than embedding OpenSearch-specific code.

Phase 14 — Redis

Use Redis for:

caching

rate limiting

concurrency counters

short-lived coordination

distributed locks only where appropriate

event/stream support where appropriate

Do not use Redis as the authoritative conversation database.

Phase 15 — Temporal Durable Execution

Flow:

API
 -> Create Run
 -> Start Temporal Workflow
 -> Agent Execution
 -> LLM Calls
 -> Tool Calls
 -> Memory
 -> Result

Use Temporal for:

durable workflows

retries

timers

recovery

activity execution

long-running jobs

A worker crash must not lose the workflow.

Phase 16 — Streaming

Use SSE or WebSockets.

Conceptual flow:

POST message
 -> run_id
 -> connect to stream
 -> receive events

Events:

run.started
message.created
llm.started
llm.delta
tool.started
tool.completed
message.completed
run.completed
run.failed

Client disconnect should not automatically kill the durable run.

Phase 17 — Idempotency

Implement idempotency for:

message submission

run creation

tool side effects

file operations

workflow starts

Use an Idempotency-Key and persist its association with the resulting operation.

Repeated requests should not create duplicate side effects.

Phase 18 — Concurrency & Backpressure

10,000 HTTP requests must not become 10,000 simultaneous Bedrock calls.

Control:

global concurrency

tenant concurrency

user concurrency

model concurrency

tool concurrency

database connections

Use:

admission control

queues

Redis counters

bounded worker pools

timeouts

rate limits

backpressure

Phase 19 — Observability

Use OpenTelemetry.

Trace:

HTTP request
 -> Agent run
 -> LLM call
 -> Tool call
 -> Database
 -> External service

Metrics:

request latency

p50/p95/p99

error rate

throughput

active runs

queue depth

LLM latency

token usage

tool latency

DB pool usage

Structured logs should include:
request_id, trace_id, tenant_id, user_id, run_id, agent_id

Do not unnecessarily log secrets or sensitive content.

Phase 20 — Security

Implement:

authentication

authorization

tenant isolation

secret management

encryption

TLS

input validation

file validation

prompt-injection defenses

tool authorization

audit logs

rate limiting

SSRF protection

SQL injection prevention

secure headers

dependency scanning

Treat the LLM as untrusted input, not as a security boundary.

Phase 21 — AWS Infrastructure

Suggested deployment:

Route 53
  -> API Gateway / ALB
  -> EKS or ECS
  -> FastAPI

Services:

RDS PostgreSQL

ElastiCache Redis

S3

OpenSearch

Temporal

Bedrock

monitoring/telemetry backend

Use Terraform.

Phase 22 — Containers

Production image requirements:

non-root user

minimal base image

health check

environment-based config

no baked-in secrets

graceful shutdown

predictable startup

Podman can be used locally while keeping OCI-compatible images.

Phase 23 — Autoscaling

Scale API and agent workers independently.

API Pods
  <- HTTP traffic

Agent Workers
  <- workflow/activity load

Scale on:

CPU

memory

request rate

queue depth

active workflows

Do not rely on CPU alone for AI workloads.

Phase 24 — Testing Strategy

Testing layers:

Unit
 -> Integration
 -> API
 -> Agent
 -> Workflow
 -> Load
 -> Stress
 -> Soak
 -> Chaos
 -> Disaster Recovery

Unit tests

Test:

domain logic

services

repositories

validation

authorization

memory policies

idempotency

tool schemas

Integration tests

Test:

PostgreSQL

Redis

S3

OpenSearch

Temporal

Bedrock mocks/stubs where appropriate

API tests

Test:

authentication

authorization

CRUD

pagination

validation

errors

idempotency

streaming

cancellation

Agent tests

Test:

tool selection

tool arguments

structured output

retrieval

memory use

failure handling

Agent evaluation

Track:

correctness

groundedness

retrieval quality

hallucination rate

tool accuracy

latency

cost

Maintain fixed evaluation datasets before changing prompts, models, retrieval, or memory behavior.

Phase 25 — Load, Stress & Soak Testing

Use k6 or Locust.

Measure:

p50/p95/p99 latency

throughput

errors

queue depth

DB connections

Redis load

Bedrock throttling

Stress testing:

increase load until degradation

identify first bottleneck

record saturation point

Soak testing:

run sustained traffic for hours

detect memory leaks

connection leaks

queue growth

stale locks

worker crashes

latency drift

Phase 26 — Chaos & Disaster Recovery

Chaos scenarios:

API pod dies

agent worker dies

database unavailable

Redis unavailable

Temporal worker dies

LLM timeout

LLM throttling

tool timeout

network interruption

S3 failure

Disaster recovery:

define RPO

define RTO

database backups

restore testing

S3 recovery

infrastructure recreation

secret recovery

workflow recovery

Test restore procedures, not just backup creation.

Phase 27 — CI/CD

Git Push
 -> Lint
 -> Type Check
 -> Unit Tests
 -> Integration Tests
 -> Build
 -> Security Scan
 -> Staging Deploy
 -> Smoke Tests
 -> Production

Consider:

rolling deployment

blue/green

canary

Recommended Implementation Order

Build in this exact dependency order:

Domain modeling

PostgreSQL schema

SQLAlchemy models

Alembic migrations

Repository layer

FastAPI structure

Authentication

Conversations

Messages

Agent runs

LLM gateway

Pydantic AI

Tools

Memory

Files

Document ingestion

RAG

Redis

Temporal

Streaming

Idempotency

Concurrency

Observability

Security

AWS infrastructure

Autoscaling

Testing

Load testing

Chaos testing

Production hardening

MVP Milestones

Milestone 1 — Data Foundation

Build:
Tenant, User, Conversation, Message, Agent, AgentRun

Deliverable: clean PostgreSQL schema + migrations.

Milestone 2 — Basic Backend

Build:
FastAPI, Auth, Conversation APIs, Message APIs, Run APIs

Deliverable: working backend without AI complexity.

Milestone 3 — First Agent

Build:
Pydantic AI, LLM Gateway, Bedrock

Deliverable: user message -> agent response.

Milestone 4 — Tools

Build:
Tool Registry, Authorization, Tool Calls

Deliverable: safe tool execution.

Milestone 5 — Memory

Build:
Memory Extraction, Storage, Retrieval

Deliverable: selected long-term user context.

Milestone 6 — Files + RAG

Build:
S3, ingestion, embeddings, OpenSearch, retriever

Deliverable: document-aware agent.

Milestone 7 — Durable Execution

Build:
Temporal, workflows, activities, retries, recovery

Deliverable: execution survives API/worker crashes.

Milestone 8 — Reliability

Build:
Streaming, idempotency, concurrency, rate limits, observability, security

Milestone 9 — Production Infrastructure

Build:
AWS, containers, ALB/API Gateway, EKS/ECS, RDS, Redis, S3, OpenSearch, Terraform, autoscaling

Milestone 10 — Production Validation

Complete:
Unit, integration, API, agent evaluation, load, stress, soak, chaos, DR

Completion Checklist

Data

Domain entities

ERD

Relationships

Constraints

Indexes

Migrations

Backend

FastAPI structure

DI

Error handling

Authentication

Authorization

Conversation APIs

Message APIs

Run APIs

Agent

LLM gateway

Bedrock adapter

Pydantic AI

Agent versioning

Tool registry

Tool authorization

Memory

Conversation memory

Execution state

Long-term memory

Memory extraction

Memory validation

Memory retrieval

RAG

File upload

S3

Document ingestion

Chunking

Embeddings

OpenSearch

Retriever

Reliability

Temporal

Idempotency

Retries

Timeouts

Concurrency limits

Rate limits

Backpressure

Cancellation

Production

OpenTelemetry

Metrics

Structured logging

Security

Terraform

Containers

Autoscaling

CI/CD

Backups

Disaster recovery

Testing

Unit

Integration

API

Agent evaluation

Load

Stress

Soak

Chaos

Recovery

Final Design Rules

PostgreSQL is the source of truth for application state.

Redis is not the primary database.

S3 stores large files; PostgreSQL stores file metadata.

FastAPI remains stateless.

Temporal owns durable long-running execution.

Agents should not directly access infrastructure.

All model calls go through an LLM gateway.

Tools require authorization and schemas.

Side-effecting tools require idempotency.

Conversation memory, execution state, and long-term memory are separate.

Do not store every message as long-term memory.

10,000 HTTP requests must not become 10,000 simultaneous LLM calls.

Apply bounded concurrency and backpressure.

Version agents, prompts, workflows, and model configurations.

Design observability before production.

Test failure recovery, not only successful execution.

Build data modeling first, backend second, agent capabilities third, reliability last.

Keep infrastructure behind abstractions so components can evolve independently.
