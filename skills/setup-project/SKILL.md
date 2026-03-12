---
name: setup-project
description: Bootstrap a full-stack Next.js + Convex + WorkOS AuthKit project with shadcn/ui theming, agent skills, and GitHub integration. Run this skill once to go from an empty folder to a ready-to-build project.
---

# Setup Project

This skill bootstraps the entire project in one pass. It checks dependencies, creates the project in the current directory, installs agent skills locally, sets up theming, creates an agent.md, and pushes to GitHub. The product-specific details (what you're building) come from the user's prompt — this skill just sets up the foundation.

## What this skill does

1. Checks that all required tools are installed
2. Creates the Next.js + Convex + WorkOS AuthKit project in the current directory
3. Installs agent skills locally (project-level, not global)
4. Installs shadcn/ui and applies a custom theme
5. Installs default shadcn components
6. Creates an `agent.md` with project commands and skill reference
7. Creates a Codex action for starting the dev server
8. Pushes the project to GitHub
9. Tells the user how to start the dev server

## Step 1: Check and install dependencies

This skill assumes it is already running inside the project folder.

Before doing anything else, check which of the following tools are available on this machine by running their version commands (e.g., `node -v`, `gh --version`):

- `node` (v18 or higher)
- `npm`
- `git`
- `gh` (GitHub CLI)
- `npx`

**If any tool is missing, install it automatically and continue.** Do not stop the skill to ask the user — just install and keep going. Use platform detection to choose the right installer:

- **macOS:** `brew install <package>` (if `brew` is available)
- **Linux (Debian/Ubuntu):** `sudo apt-get install -y <package>` or `curl`-based install
- **Linux (other):** `curl`-based install from the official source
- **Windows:** `winget install <package>` or `choco install <package>` (if available)

Special cases:
- **Node.js + npm (missing on macOS):** Run `brew install node`. If `brew` is not available, tell the user to download from https://nodejs.org and wait for them to confirm before continuing — this is the only case where you must pause.
- **Node.js + npm (missing on Linux):** Use `curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -` then `sudo apt-get install -y nodejs`, or direct the user to https://nodejs.org.
- **Node.js + npm (missing on Windows):** Use `winget install OpenJS.NodeJS.LTS` or direct the user to https://nodejs.org.
- **git (missing):** macOS: `brew install git` or `xcode-select --install`. Linux: `sudo apt-get install -y git`. Windows: `winget install Git.Git`.
- **gh (missing):** macOS: `brew install gh`. Linux: see https://github.com/cli/cli/blob/trunk/docs/install_linux.md. Windows: `winget install GitHub.cli`.

After all tools are confirmed present, check if `gh` is authenticated:

```bash
gh auth status
```

If `gh` is **not authenticated**, tell the user:

> "I need you to sign into GitHub from the terminal. This is going to open your browser — sign in and come back here."

Then run:

```bash
gh auth login
```

This is interactive — it opens the browser for GitHub authorization. Wait for the user to complete it before continuing.

Do not proceed to the next step until all tools are installed and `gh` is authenticated.

## Step 2: Create the project

**Before scaffolding**, check if the current directory already has files. If it does:

1. Move existing files to a temporary backup location (e.g., `/tmp/project-backup-<timestamp>/`)
2. Run the scaffold command
3. Restore the backed-up files on top of the scaffolded project (without overwriting scaffolded files)

Run this exact command to scaffold into the current directory (non-interactive — no menus):

```bash
npm create convex@latest . -- -t nextjs-authkit
```

The `.` tells it to create the project in the current directory instead of making a new folder.

**Note:** The Convex browser auth (signing in with GitHub to create a Convex account) happens when the user starts the dev server for the first time — not during this scaffold step. If the scaffold command opens a browser for Convex auth, tell the user:

> "Convex is asking you to sign in with GitHub. This creates your free Convex account and also sets up the sign-in system automatically. Sign in and come back here."

Wait for the user to confirm they've signed in before continuing.

## Step 3: Install agent skills

Install coding skills **locally** into the project. These stay in the project's skill directories and don't affect other projects.

Use the `-a` flag to install skills for both Codex and Claude Code:

```bash
npx skills add anthropics/skills --skill frontend-design -a codex -a claude-code -y
npx skills add vercel-labs/agent-skills --skill web-design-guidelines -a codex -a claude-code -y
npx skills add vercel-labs/agent-skills --skill vercel-react-best-practices -a codex -a claude-code -y
npx skills add waynesutton/convexskills --skill convex -a codex -a claude-code -y
npx skills add get-convex/agent-skills --skill convex-helpers-guide -a codex -a claude-code -y
npx skills add workos/skills --skill workos -a codex -a claude-code -y
npx skills add shadcn/ui --skill shadcn -a codex -a claude-code -y
```

These give the agent knowledge about:
- **frontend-design** — production-grade UI that avoids generic "AI slop" aesthetics
- **web-design-guidelines** — web design fundamentals and accessibility
- **vercel-react-best-practices** — React and Next.js patterns, RSC, data fetching
- **convex** — Convex schema, queries, mutations, and real-time patterns
- **convex-helpers-guide** — Convex helper utilities and advanced patterns
- **workos** — WorkOS AuthKit integration patterns
- **shadcn** — shadcn/ui component usage and theming

## Step 4: Set up shadcn/ui

Stop and ask the user:

> "Time to pick your theme. Open https://ui.shadcn.com/create in your browser, customize the colors and style you want, then give me:
> - The CLI command it gives you, or
> - The CSS variables to paste into your styles
>
> I'll apply it for you."

Wait for the user to provide the theme command or CSS. Then:

- If they give a CLI command: run it
- If they give CSS variables: paste them into `app/globals.css` (or wherever the project's global styles live), replacing the existing `:root` and `.dark` color variable blocks

If the CLI command already initializes shadcn (e.g., it includes `init`), skip any separate init step. If it only applies a theme, run `npx shadcn@latest init -d -y` first, then apply the theme.

## Step 5: Install shadcn components

Run this command (non-interactive):

```bash
npx shadcn@latest add button card input textarea badge dialog -y -o
```

These are the base components most projects need. Infer additional components needed based on the product plan or feature requirements described by the user, and install them too.

## Step 6: Create agent.md

Create a file called `agent.md` at the project root. Infer the project name from the folder name or the user's description. Use this structure:

```markdown
# [Project Name] — Agent Guide

## Running the project

Start both the Next.js frontend and Convex backend with one command:

\`\`\`bash
npm run dev
\`\`\`

This starts:
- Next.js dev server at `http://localhost:3000`
- Convex dev server syncing your backend functions

## Validating changes

After every code change, run this command to check that the Convex schema and functions compile without errors:

\`\`\`bash
npx convex dev --once
\`\`\`

This runs the Convex compiler once and exits. If there are errors, fix them before moving on. Do not skip this step.

## Codex action

To configure the dev server as a Codex action: open the Codex app settings, go to Actions, and add a new action with the command `npm run dev`. This adds a play button to the Codex toolbar for one-click server start.

When the dev server starts for the first time, Convex will open your browser to sign in with GitHub — this creates your free Convex account and sets up the sign-in system.

## Installed skills

This project has the following agent skills installed:

| Skill | What it teaches |
|-------|----------------|
| **frontend-design** | Production-grade UI — avoid generic aesthetics, use intentional design choices |
| **web-design-guidelines** | Web design fundamentals, accessibility, responsive patterns |
| **vercel-react-best-practices** | React and Next.js patterns — RSC boundaries, data fetching, App Router conventions |
| **convex** | Convex schema design, queries, mutations, real-time subscriptions |
| **convex-helpers-guide** | Convex helper utilities and advanced patterns |
| **workos** | WorkOS AuthKit integration — sign-in, sign-out, user identity |
| **shadcn** | shadcn/ui component usage, theming, and composition |

When building features, follow the patterns described in these skills. They are your reference for how to write code in this project.

## Stack

| Layer | Tool |
|-------|------|
| Frontend | Next.js (App Router) |
| Styling | Tailwind CSS |
| Components | shadcn/ui |
| Database | Convex |
| Auth | WorkOS via Convex AuthKit |
| Deployment | Vercel |

## Key commands

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start the full dev environment (frontend + backend) |
| `npx convex dev --once` | Validate Convex schema and functions compile |
| `npx convex dashboard` | Open the Convex dashboard in the browser |
| `npx shadcn@latest add [component] -y -o` | Add a shadcn component |
```

## Step 7: Push to GitHub

Now that the project exists and we're authenticated with `gh`, create a GitHub repo and push:

```bash
git init
git add -A
git commit -m "Initial project setup — Next.js + Convex + WorkOS AuthKit"
gh repo create <project-name> --private --source=. --push
```

Infer the repo name from the project folder name. Use `--private` to keep the repo private by default.

The `gh repo create` command creates the repo and pushes in one shot — no interactive prompts.

Tell the user:

> "Your code is now on GitHub. Every change we make from here gets saved there."

## Step 8: Hand off

Tell the user:

> "The project is set up and ready to build. Start the dev server by running `npm run dev` in your terminal, or by clicking the play button in the Codex GUI.
>
> When the dev server starts for the first time, Convex will open your browser to sign in with GitHub — this creates your free Convex account and sets up the sign-in system. Sign in and come back here."

Do not mention Vercel, deployment, or any future steps. The user is ready to start building features.
