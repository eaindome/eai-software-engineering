# eai-software-engineering

A personal knowledge base for software engineering practices, review methodology, and stack-specific patterns. Built from real project experience.

## Contents

**`ENGINEERING_REVIEW.md`** — The core engineering review playbook. Stack-agnostic. Covers how to review a feature across five passes: implementation defects, logical correctness, system-wide impact, assumptions and gaps, and failure recovery. Includes Phase 0 (feature map) for structuring multi-feature reviews so they don't go shallow.

**`stacks/`** — Stack-specific companion files. Each one plugs into Pass 1 of the core playbook and covers the gotchas, anti-patterns, and conventions specific to that tech combination.

- `stack-cai-dotnet-graphql.md` — Cai framework / .NET / HotChocolate GraphQL / EF Core

## How to use

For a new codebase review, start with `ENGINEERING_REVIEW.md`. Run Phase 0 first to map and sequence features with a human reviewer. Then review one feature at a time using the five-pass structure. If a matching stack companion exists, apply it alongside Pass 1.

For a PR review or incident follow-up, skip Phase 0 and go straight to the five passes on the relevant scope.

## Principles

The core playbook never changes stacks. If you move to a different stack, add a new companion file — don't modify `ENGINEERING_REVIEW.md`. The methodology is durable; the stack knowledge is modular.
