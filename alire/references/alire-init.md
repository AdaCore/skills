# Creating a Project

Collect from the user before proceeding:
- **Project name** — lowercase, alphanumeric, underscores allowed (e.g. `my_project`)
- **Project description** — a short one-line description
- **Project type** — `Binary` (default) or `Library`

Initialize in non-interactive mode:
```bash
alr -n init --bin <project_name>   # binary
alr -n init --lib <project_name>   # library
```

Update the `description` field in `<project_name>/alire.toml` using the Edit tool, then verify:
```bash
cd <project_name> && alr -n build
```
