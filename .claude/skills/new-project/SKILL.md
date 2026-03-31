---
name: new-project
description: "Create a new org-mode TODO entry and project folder. Use when the user wants to start a new project, add a TODO with a project folder, or says 'new project'."
argument-hint: "<project description>"
---

# New Project

Creates a TODO entry in `~/org/todos.org` and a corresponding project folder via the `nb-new` Emacs function.

## Arguments

The user provides the project description: $ARGUMENTS

## Instructions

### Step 1: Get project description

If `$ARGUMENTS` is empty, ask the user for the project description.

### Step 2: Ask for category and deadline

Read `~/org/todos.org` to find the top-level `* Category` headings (e.g. Work, Emacs, Sport, Home, "When time permits"). Use AskUserQuestion to ask the user:

1. **Which category?** — present the categories found as options
2. **What deadline?** — ask for a date (suggest today's date as default)

### Step 3: Add TODO entry to todos.org

Insert a new entry under the chosen category heading in `~/org/todos.org`:

```org
** TODO <project description>
DEADLINE: <YYYY-MM-DD Day>
```

The date format must match org-mode style, e.g. `<2026-03-28 Sat>`. Insert it **after existing TODO/STRT/WAIT items** but **before any DONE/CNCL items** in that category section. If there are no DONE/CNCL items, append at the end of the category section (before the next `* ` heading).

### Step 4: Create project folder

Compute the folder name: take the deadline date (YYYY-MM-DD) and the project description converted to kebab-case (lowercase, spaces/underscores become hyphens, remove special characters except hyphens).

Example: deadline `2026-03-28`, description "Test drawing with Haskell diagrams" → `2026-03-28-test-drawing-with-haskell-diagrams`

Create the folder via emacs --batch:

```bash
emacs --batch -l ~/my-guix-profiles/emacs/config/.emacs --eval '(nb-new "~/org/projects/YYYY-MM-DD-kebab-case-name")'
```

If this fails, do NOT create the folder manually. Instead, tell the user the folder creation failed and suggest they check their Emacs setup.

### Step 5: Add emacs environment

Copy the emacs project template into the new project folder:

```bash
cp -r ~/my-guix-profiles/emacs-project-template/ <project-folder>/emacs/
```

This gives the project its own Guix-managed Emacs environment that can be started from anywhere via `<project-folder>/emacs/emacs` and supports `--daemon=<name>` for server mode.

### Step 6: Configure local Claude settings

Create `.claude/settings.local.json` in the new project folder with auto-memory pointed at the project directory (so it syncs via Syncthing):

```json
{
  "autoMemoryDirectory": ".claude/memory"
}
```

### Step 7: Write README.org

Write a `README.org` file in the new project folder:

```org
#+TITLE: <Project Description in Title Case>

<Write a 1-2 sentence description of what this project is about, based on the project name.>
```

### Step 8: Confirm

Tell the user:
- The TODO was added to `~/org/todos.org` under the chosen category
- The project folder was created at the full path
