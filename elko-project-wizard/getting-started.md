# Getting Started — elko-project-wizard

## 1. Pull and Run

```bash
docker pull jsoprych/elko-project-wizard:latest

docker run -d \
  --name elko-project-wizard \
  -p 8080:8080 \
  -v wizard-data:/app/data \
  jsoprych/elko-project-wizard:latest
```

Open **http://localhost:8080** — the dashboard loads immediately.

---

## 2. Use the Quick Start Wizard

If this is your first time, click **🚀 Quick Start Wizard** in the sidebar.

The wizard walks you through **4 color-coded tabs**, left to right. Each tab must be completed before **Next →** activates — a warning appears if you try to skip ahead.

1. **🛠️ Tech Stack** (blue) — pick Go, Node.js/TypeScript, Python, Rust, LAMP, C, C++
2. **🎓 Experience** (green) — Zero (beginner), Builder (intermediate), Hero (expert)
3. **📁 Project** (orange) — enter a name, e.g. `my-api`
4. **⚡ Generate** (purple) — review the bundle, click **⬇ Generate & Download ZIP**

Tabs show **✓** when complete. You can click any tab to jump back and change a selection.

---

## 3. Use Your Scaffold

```bash
unzip my-api.zip
cd my-api
git init
# Open Claude Code or OpenCode — AGENTS.md already briefs the agent
claude
```

Your AI agent reads `AGENTS.md` (via `CLAUDE.md → @AGENTS.md`) and knows your stack, rules, and preferences before writing a single line.

---

## 4. Save a Profile for Next Time

Click **+ New Profile** in the sidebar, select your directives, name it (e.g. *Go Private Greenfield*), and save. Next project — one click to generate.

→ [Profiles guide](profiles.md)

---

## Stop / Remove

```bash
docker stop elko-project-wizard
docker start elko-project-wizard
docker rm elko-project-wizard                                        # remove container (data kept)
docker rm elko-project-wizard && docker volume rm wizard-data        # full removal
```

---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
