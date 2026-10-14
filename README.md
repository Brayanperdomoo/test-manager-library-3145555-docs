# Manager Library — Specification

**Technical assessment · ADSO cohort 3145555**
**Schedule:** Wednesday, October 14, 2026, 6:50 a.m.–11:00 a.m. Colombia time. **Submission deadline:** 11:00 a.m.

This repository is the source of truth for the Manager Library product requirements. Its business rules, acceptance criteria, data model, and HTTP contract apply to all implementations.

## Assignment

Create a fork of all six project repositories and implement the system in the required stack: **React** for the frontend and **PHP with Laravel** for the backend. The reference specifications are written for Python and .NET; their business requirements must be applied to the challenge stack.

The assessment focuses on **Spec-Driven Development (SDD)**: reading, interpreting, and applying a specification when moving to a different technology stack. Code is the means, not the end. Push incremental, reviewable work before the deadline; late submissions are not reviewed.

## Specification folders

| Folder | Reference implementation |
|---|---|
| [`spec-python/`](spec-python/) | Python backend reference |
| [`spec-.net/`](spec-.net/) | .NET backend reference |

Both versions describe the same product and must remain aligned on business behavior. Their technology choices are references only; the assignment target is React and Laravel.

## Reading order

Within either folder, use this order:

1. `constitution.md` — non-negotiable principles.
2. `spec.md` — actors, stories, acceptance criteria, and business rules.
3. `plan.md`, `architecture.md`, and `arquitectura-panoramica.md` — reference implementation approach.
4. `data-model.md` and `api-contract.md` — persistence model and HTTP contract.
5. `tasks.md` — work items and completion evidence.
6. `adr/` — architecture decisions, alternatives, and consequences.

## Project repositories

| Repository | Responsibility |
|---|---|
| `test-manager-library-3145555-docs` | Product specification and contracts |
| `test-manager-library-3145555-api` | Laravel API |
| `test-manager-library-3145555-app` | React operations interface |
| `test-manager-library-3145555-page` | Static public presentation site |
| `test-manager-library-3145555-infra` | Containers, network, volumes, and empty database |
| `test-manager-library-3145555-tool` | Demo-data seeder using the API |

All project documentation and application-facing text for this implementation are in English. Keep business behavior consistent with the corrected specification.
