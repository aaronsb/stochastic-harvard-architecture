# Stochastic Harvard Architecture — Project Instructions

## What This Is

An exploratory specification for applying Harvard architecture (separate instruction/data buses) to AI agent systems. The core insight: prompt injection is a structural von Neumann problem, not a bug.

## Project Structure

```
docs/
├── 00-overview.md through 12-glossary.md   # Numbered spec sections
├── architecture/                            # ADRs (design decisions)
│   └── adr.yaml                            # ADR domain config
└── scripts/
    └── adr                                  # ADR CLI tool
.claude/
└── ways/sha/                               # Project-local ways
```

## Conventions

- SHA terminology is defined in docs/12-glossary.md — use it consistently
- Diagrams use Mermaid with the project color palette (teal/purple/green/orange/blue)
- ADR domains: core (100s), bus (200s), security (300s), implementation (400s)
- Spec sections are numbered 00-12 in docs/ with prev/next navigation
