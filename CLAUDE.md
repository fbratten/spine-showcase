# CLAUDE.md - SPINE Showcase

> **Project:** spine-showcase
> **Type:** Static GitHub Pages Site (Showcase/Documentation)
> **Status:** Active
> **Created:** 2025-12-13
> **GitHub:** https://github.com/fbratten/spine-showcase
> **Live Site:** https://fbratten.github.io/spine-showcase/

---

## Quick Context

This is a **static showcase site** for SPINE (Structured Pipeline for Intelligent Node Execution). It demonstrates SPINE's capabilities through interactive demos and documentation.

**Project Type:** Static HTML/CSS/JS (no backend, no build process)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Hosting | GitHub Pages |
| Styling | Tailwind CSS (CDN) |
| Fonts | Google Fonts (Inter) |
| Build | None (static files) |

---

## Project Structure

```
spine-showcase/
├── index.html              # Main landing (redirects to README on GitHub)
├── README.md               # Main documentation
├── KB/                     # Knowledge base
│   └── Multi-Agent-Playbook-Blueprint.pdf
├── demos/                  # Interactive demonstrations
│   ├── index.html          # Demo gallery
│   ├── multi-agent-playbook/   # Blueprint viewer
│   ├── tier-classifier/    # Tier classification demo
│   ├── provider-picker/    # LLM provider selector
│   ├── cost-calculator/    # Cost estimation tool
│   ├── fan-out-simulator/  # Parallel execution demo
│   └── pipeline-builder/   # Sequential pipeline demo
├── docs/                   # Documentation
│   ├── README.md
│   ├── architecture.md
│   ├── patterns.md
│   ├── tiered-enforcement.md
│   └── blueprint-implementation.md
└── screenshots/            # Demo screenshots
```

---

## SPINE Usage Protocol

Since this is a **static showcase site**, full SPINE orchestration is not applicable.

**Tier Classification:**
- Most tasks are **Tier 1** (single-file HTML/CSS edits)
- Multi-file documentation updates may be **Tier 2**

**Applicable Capabilities:**
- `Explore` subagent for understanding existing patterns
- `browser-mcp` for visual verification of demos
- Standard git workflow for commits

---

## Development Notes

### Adding New Demos

1. Create folder in `demos/`
2. Add `index.html` with Tailwind CSS CDN
3. Update `demos/index.html` gallery
4. Update README.md demo count badge

### Updating Documentation

1. Edit markdown files in `docs/`
2. Ensure cross-references are updated
3. Commit with descriptive message

### PDF References

The Multi-Agent Playbook PDF is stored in `KB/` folder.
When referencing:
- From root: `KB/Multi-Agent-Playbook-Blueprint.pdf`
- From demos: `../../KB/Multi-Agent-Playbook-Blueprint.pdf`
- From docs: `../KB/Multi-Agent-Playbook-Blueprint.pdf`

---

## Git Workflow

```bash
# Standard workflow
git add .
git commit -m "Description of changes"
git push origin main

# GitHub Pages auto-deploys from main branch
```

---

## Related Projects

| Project | Relationship |
|---------|--------------|
| SPINE | Core framework (read-only reference) |
| spine-dashboard | Monitoring UI for SPINE |

---

## References

- **SPINE Protocols:** `/mnt/d/temp/myprojects/spine/_protocols/`
- **SPINE Templates:** `/mnt/d/temp/myprojects/spine/_templates/`
- **Live Site:** https://fbratten.github.io/spine-showcase/

---

*Document created: 2025-12-30*
