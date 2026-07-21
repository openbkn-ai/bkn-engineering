# OpenBKN Engineering

[中文](README.zh.md) | English

OpenBKN Engineering provides Agent Skills and methodology for building Business Knowledge Networks (BKNs) with a repeatable engineering workflow.

It helps FDEs, AI engineers, product teams, and delivery teams move from business material to a validated, operable knowledge network:

```text
Business material / interviews / PRD drafts
  -> bkn-requirement
  -> scenario-centered PRD + BKN Creator handoff
  -> bkn-ontology-builder
  -> business-reviewable ontology modeling scheme
  -> bkn-creator
  -> BKN modeling / binding / testing / validation / publishing
  -> feedback review
  -> delivery archive
```

The goal is to make Agent work traceable, reviewable, and reusable. Each stage has a clear input, output, handoff boundary, and verification gate.

## Skills

This repository publishes four top-level skills.

| Skill | Role | Main outputs | Out of scope |
|---|---|---|---|
| `bkn-requirement` | Requirement discovery and PRD structuring | Research outlines, meeting digests, scenario-centered PRDs, acceptance cases, BKN Creator handoff summaries | Does not create `.bkn` files, bind data, or publish platform resources |
| `bkn-ontology-builder` | Ontology modeling scheme generation and refinement | Business-reviewable ontology modeling schemes, verifier findings, final gate reports, implementation-feedback revisions | Does not create `.bkn` files, bind data, or publish platform resources |
| `bkn-methodology` | Shared BKN modeling method and review rules | Object / relation / fact / metric / action / governance boundary guidance | Does not run project workflows by itself |
| `bkn-creator` | BKN lifecycle orchestration | BKN creation, extraction, update, copy, validation, binding, reports, feedback review, delivery archive | Not for pure semantic data query |

## Recommended Workflow

### 1. Discover Requirements

Use `bkn-requirement` when you need to turn customer background, interviews, meeting notes, PRDs, BRDs, process descriptions, system material, or data material into a business-readable PRD.

Example:

```text
Use $bkn-requirement to turn these interview notes into a scenario-centered PRD and BKN Creator handoff summary.
```

### 2. Design The Ontology

Use `bkn-ontology-builder` when you need a business-reviewable ontology modeling scheme before formal BKN creation.

Example:

```text
Use $bkn-ontology-builder to generate an ontology modeling scheme from this PRD and handoff summary.
```

### 3. Apply The Methodology

Use `bkn-methodology` as the common rule base for deciding object boundaries, relation semantics, facts, metrics, operators, actions, governance points, and Skill / Agent responsibilities.

Example:

```text
Use $bkn-methodology to review whether these candidate objects and relations are valid BKN modeling choices.
```

### 4. Create And Validate The BKN

Use `bkn-creator` when the workflow enters BKN creation, extraction, update, binding, validation, feedback review, or delivery archive.

Example:

```text
Use $bkn-creator to create a BKN from this ontology modeling scheme. Show the route and execution preview first, then wait for confirmation.
```

## Install

Install a selected skill with `npx skills`:

```bash
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-requirement
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-ontology-builder
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-methodology
npx skills add https://github.com/openbkn-ai/bkn-engineering --skill bkn-creator
```

Restart your agent session after installation so the skill list refreshes.

Platform operations such as authentication, validation, push / pull, Trace, Eval, and admin tasks are provided by `@openbkn/bkn-sdk`:

```bash
npm install -g @openbkn/bkn-sdk
openbkn --help
```

## Repository Layout

```text
bkn-engineering/
  README.md
  README.zh.md
  LICENSE
  NOTICE
  skills/
    bkn-requirement/
      SKILL.md
      agents/
      assets/
      references/
    bkn-ontology-builder/
      SKILL.md
      agents/
      assets/
      references/
    bkn-methodology/
      SKILL.md
      references/
    bkn-creator/
      SKILL.md
      internal/
        _pipelines/
        _plugins/
        _shared/
        bkn-archive/
        bkn-backfill/
        bkn-bind/
        bkn-doctor/
        bkn-domain/
        bkn-draft/
        bkn-env/
        bkn-extract/
        bkn-map/
        bkn-openbkn/
        bkn-relation-bind/
        bkn-report/
        bkn-review/
        bkn-skillgen/
        references/
```

Each published skill must be self-contained. A skill should reference files inside its own directory at runtime.

## BKN Creator Pipelines

`bkn-creator` routes user intent to lifecycle pipelines.

| Intent | Pipeline |
|---|---|
| Create a BKN | `internal/_pipelines/create.md` |
| Extract BKN candidates from documents | `internal/_pipelines/extract.md` |
| Read or inspect BKN assets | `internal/_pipelines/read.md` |
| Update or rebind a BKN | `internal/_pipelines/update.md` |
| Delete BKN assets | `internal/_pipelines/delete.md` |
| Copy or clone a BKN | `internal/_pipelines/copy.md` |
| Validate and diagnose a BKN | `internal/_pipelines/validate.md` |
| Generate Skill drafts | `internal/_pipelines/skill-gen.md` |
| Review feedback and improve a BKN | `internal/_pipelines/feedback.md` |

All pipelines follow the same gate sequence:

```text
discovery -> preview -> confirm -> execute -> verify -> report
```

Write operations run only after explicit confirmation.

## Project Archive Convention

For customer or project work, keep source material and generated deliverables in one project folder:

```text
projects/prj-<project-name>/
  inputs/
    round-01/
      source-manifest.md
  <project>-PRD-v0.1.md
  <project>-ontology-modeling-scheme-v0.1.md
  bkn/
    network.bkn
    object_types/
    relation_types/
    action_types/
  delivery/
    validation-report.md
    feedback-review.md
    final-archive.md
```

Do not move original user files. If external source files are used, copy or register them under the current round and record them in `source-manifest.md`.

## Development Guidelines

- Keep public docs user-friendly: explain what the project does, when to use each skill, how to install it, and what output to expect.
- Keep the four skills at the same top-level under `skills/`.
- Keep each skill self-contained for independent installation.
- Keep platform operations delegated to `@openbkn/bkn-sdk`.
- Require explicit confirmation before platform write operations.
- Update `NOTICE` when external material is included.

## License

OpenBKN Engineering is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
