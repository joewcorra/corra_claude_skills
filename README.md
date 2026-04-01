# Claude Skills Collection

A curated collection of helpful custom skills for Claude AI, designed to extend Claude's capabilities with specialized functionality.

## Overview

This repository contains reusable Claude Skills that provide enhanced functionality for various AI tasks. Each skill is self-contained and documented, making it easy to integrate into your Claude workspace.

## Skills

### 📊 R Methodology Narrator
**Directory:** `r-methodology-narrator/`

A skill that provides narrative explanations of R methodologies. This skill helps Claude deliver comprehensive, methodology-focused insights for research-related tasks. This skill directs Claude to translate completed R modules back into plain-language methodology documentation.

[Learn more →](./r-methodology-narrator/)

### 🧬 R GHG Coder
**Directory:** `r-ghg-coder/`

An AI coding skill for the U.S. Greenhouse Gas Inventory. Converts structured implementation frameworks — as produced by the `ghg-methodology-parser` skill — into production-ready R code following the project's established conventions. Output is a complete, `{targets}`-pipeline-ready R script composed of modular `get_` functions, one per logical calculation phase, ready to drop into the inventory codebase.

This skill is purpose-built for the U.S. GHG Inventory. Its conventions — function naming, constants vector structure, `{syrinx}` integration, CRF source category organization, and GHG-specific column naming — reflect that context directly. The underlying architecture (one function per calculation phase, named constants vectors, canonical units at ingestion, `{targets}`-compatible composition) is methodology-agnostic and could be adapted to other large-scale reproducible R pipelines with relatively minor modification.

#### Skill Workflow

| Skill | Role |
|---|---|
| `ghg-methodology-parser` | Translates narrative inventory methodology into a structured, language-agnostic implementation framework |
| `r-ghg-coder` | Converts that framework into production R code |
| `r-methodology-narrator` | Translates completed R modules back into plain-language methodology documentation. Ideal for reviewing step-by-step process |

[Learn more →](./r-ghg-coder/)

## Structure

Each skill is organized in its own directory with:
- `SKILL.md` - Complete skill documentation and instructions
- Additional supporting files as needed

## Getting Started

1. Browse the skills directory to find what you need
2. Read the `SKILL.md` file in each skill's folder
3. Copy the skill instructions into your Claude prompts or integration
4. Customize as needed for your use case

## Contributing

Have ideas for new skills or improvements? Feel free to contribute!

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**Created by:** [@joewcorra](https://github.com/joewcorra)  
**Last Updated:** April 1, 2026
