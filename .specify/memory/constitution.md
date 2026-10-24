# AI Work Todo Constitution

## Core Principles

### I. Spec Is the Source of Truth
Every plan, task, and line of code traces back to a requirement in `specs/<feature>/spec.md`.
A plan or implementation never silently changes a requirement; a gap found while planning is
recorded and raised, and the spec is amended through the spec workflow.

### II. Small, Boring, Low-Cost
This is a personal side project. Prefer well-supported, widely used tools and managed services
with a free or near-free tier. Add a component, service, or abstraction only when a current
requirement needs it; every addition is justified in the plan's Complexity Tracking.

### III. Private by Default, User in Control
User data is visible only to its owner. Work information leaves the system for an AI provider only
on an explicit user request and after the user has been told what is sent. AI output is always a
proposal: nothing is created, moved, or deleted until the user confirms it.

### IV. Independently Testable Increments
Each user story is deliverable and verifiable on its own, in priority order. Data-access rules and
AI output contracts are covered by automated tests; the app stays fully usable when AI is down.

### V. Public Repository Hygiene
The repository is public. No secrets, keys, tokens, personal data, or private local paths are
committed; configuration is by environment variables and a documented example file.

## Additional Constraints

- Technology choices are written down with the alternatives considered, so they can be overruled.
- Third-party service limits and prices cited in a plan carry a source and a date.
- Visual and screen design is a separate step from the technical plan.

## Development Workflow

- Work flows spec → plan → tasks → implementation, one reviewed pull request per step.
- Commits follow Conventional Commits.
- A pull request states which spec requirements it covers and any open decisions.

## Governance

This constitution is deliberately short. It is amended by a pull request that explains the change;
the version follows semantic versioning (MAJOR: principle removed or redefined, MINOR: principle or
section added, PATCH: wording). Plans check themselves against it, and reviewers may ask for a
justification in Complexity Tracking when a principle is bent.

**Version**: 1.0.0 | **Ratified**: 2026-10-24 | **Last Amended**: 2026-10-24
