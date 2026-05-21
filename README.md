

# Speckit Workflow

A comparison and integration guide for **Speckit** and **SQUAD** — two complementary AI-assisted development tools.

---

## References

| Tool | Repository |
|------|-----------|
| spec-kit | https://github.com/github/spec-kit |
| squad | https://github.com/bradygaster/squad |

---

## Speckit Flow

```
Constitution → Specify → Plan → Tasks → Implement
```

---

## Default Project Structure

```
project-root/
├── Constitution.md
└── .specify/
    ├── scripts/
    └── prompts/
```

---

## Installation

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
```

## Project Setup

```bash
specify init --integration copilot --scripts ps
```

---

## Speckit Commands

### speckit.constitution
```
/speckit.constitution create Python FastAPI application
```

### speckit.specify
```
/speckit.specify Build a TODO API that allows users to create, read, update, and
delete todo items. Each todo has a title, description, due date, and completion
status. Todos can be filtered by status and sorted by due date.
```

### speckit.plan
```
/speckit.plan use FastAPI with SQLAlchemy and SQLite for storage. Structure as a
single project with vertical slice architecture. Use pytest for testing.
```

### speckit.tasks
```
/speckit.tasks
```

**Task loading — design documents loaded in parallel:**

| Document | Lines |
|----------|-------|
| `spec.md` | full |
| `plan.md` | full |
| `data-model.md` | full |
| `todo-api.md` | 1–200 |
| `tasks-template.md` | 1–253 |

> Note: The `before_tasks` hook is optional.

---

## SQUAD

**Repository:** https://github.com/bradygaster/squad

A human-directed team of AI agents. SQUAD puts you and your development team in charge of a multi-agent workflow.

### Project Structure

```
.squad/
├── team.md
├── decisions.md
└── agents/
    ├── rusty/        # Lead agent
    ├── basher/       # Architect
    ├── linus/        # Developer
    └── livingston/   # Reviewer
```

### Installation

```bash
npm install -g @bradygaster/squad-cli@latest
```
