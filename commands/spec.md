---
description: Create or update a feature specification (PRD/SDD)
agent: build
subtask: true
---

Create or update a specification for the given feature following spec-driven development.

## User Input

$ARGUMENTS

## Process

1. **Interpret request**: Parse what feature the user wants specified.

2. **Load methodology**: Load the `prd` and `spec-driven-development` skills.

3. **Ask clarifying questions**: If ambiguous, ask up to 3 targeted questions:
   - What is the core problem being solved?
   - Who are the users/actors?
   - What are the success criteria?

4. **Write spec**: Create or update a spec file under `tasks/` with:
   - Feature name and brief description
   - Actors and user stories
   - Functional requirements
   - Non-functional constraints
   - Success criteria
   - Out of scope (if applicable)

5. **Present**: Show the spec summary and ask for approval.

6. **Save**: Write to `tasks/{feature-name}/spec.md`.

7. **Next steps**: Suggest follow-up commands like `/plan` or implementation tasks.
