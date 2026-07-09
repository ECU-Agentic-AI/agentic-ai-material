# Microsoft Agent Framework Document Series

This document tracks the Microsoft Agent Framework teaching sequence for the ECU Applied Agentic AI course.

## Scope and assumptions

- All examples use **.NET 10**
- All examples use **Microsoft Agent Framework**
- Local runtime examples assume **Foundry Local exposes an OpenAI-compatible REST interface**
- C# examples use the **OpenAI-compatible endpoint pattern**, which means the same agent server design works with Foundry Local and other compatible runtimes by changing configuration
- Aspire is used to manage the distributed services around the agent application

## Why this architecture matters

Students need to separate three concerns:

1. **Runtime** - Foundry Local hosts the model and exposes HTTP endpoints
2. **Framework** - Agent Framework provides the agent abstraction, orchestration surface, and tool integration
3. **Application** - your client, server, policies, observability, and business logic

That distinction makes it easier to swap runtimes, compare providers, and understand what part of the system is responsible for behavior.

## Planned document set

1. [01-foundry-local-core-agent-concepts.md](01-foundry-local-core-agent-concepts.md)
2. `02-building-agent-tools-in-csharp-local.md`
3. `03-local-memory-rag-and-knowledge.md`
4. `04-local-multi-agent-orchestration.md`
5. `05-local-observability-and-evaluation.md`
6. `06-local-security-and-governance.md`
7. `07-local-performance-and-cost-control.md`
8. `08-semantic-kernel-and-autogen-with-foundry-local.md`
9. `09-foundry-local-reference-architecture-and-prod-path.md`

## Current status

- Document 1 is now saved in this repository.
- Remaining documents are pending your review and direction.