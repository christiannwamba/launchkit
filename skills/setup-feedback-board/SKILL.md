---
name: setup-feedback-board
description: Bootstrap a customer feedback board project with Next.js, Convex, WorkOS AuthKit, shadcn/ui theming, and all required dependencies. Run this skill once to go from an empty folder to a running dev environment.
---

# Setup Feedback Board

This skill bootstraps the entire project in one pass. It checks dependencies, creates the project in the current directory, installs agent skills locally, sets up theming, creates an agent.md, pushes to GitHub, and starts the dev server.

## What this skill does

1. Checks that all required tools are installed
2. Creates the Next.js + Convex + WorkOS AuthKit project in the current directory
3. Pushes the initial project to GitHub immediately
4. Installs agent skills locally (project-level, not global)
5. Installs shadcn/ui and applies a custom theme
6. Installs shadcn components needed for the feedback board
7. Creates an `agent.md` with project commands and skill reference
8. Verifies everything works

## Step 1: Check and install dependencies

This skill assumes it is already running inside the project folder (e.g., `feedback-board/`).

Before doing anything else, check which of the following tools are available on this machine by running their version commands (e.g., `node -v`, `gh --version`):

- `node` (v18 or higher)
- `npm`
- `git`
- `gh` (GitHub CLI)
- `npx`

**If any tool is missing, install it automatically and continue.** Do not stop the skill to ask the user — just install and keep going:

- **Node.js + npm (missing):** Run `brew install node` (macOS). If `brew` is not available, tell the user to download from https://nodejs.org and wait for them to confirm before continuing — this is the only case where you must pause.
- **git (missing):** Run `brew install git` or `xcode-select --install`
- **gh (missing):** Run `brew install gh`

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

The current directory is the project folder. Run this exact command to scaffold into the current directory (non-interactive — no menus):

```bash
npm create convex@latest . -- -t nextjs-authkit
```

The `.` tells it to create the project in the current directory instead of making a new folder.

**Important:** When Convex opens the browser to sign in with GitHub, tell the user:

> "Convex is asking you to sign in with GitHub. This creates your free Convex account and also sets up WorkOS for sign-in automatically. Sign in and come back here."

Wait for the user to confirm they've signed in before continuing.

## Step 3: Push to GitHub

Now that the project exists and we're authenticated with `gh`, create a GitHub repo and push immediately:

```bash
git init
git add -A
git commit -m "Initial project setup — Next.js + Convex + WorkOS AuthKit"
gh repo create feedback-board --public --source=. --push
```

The `gh repo create` command creates the repo and pushes in one shot — no interactive prompts.

Tell the user:

> "Your code is now on GitHub. Every change we make from here gets saved there."

## Step 4: Install agent skills

Install coding skills **locally** into the project (not globally — no `-g` flag). These stay in the project's `.claude/skills/` directory and don't affect other projects:

```bash
npx skills add anthropics/skills --skill frontend-design -y
npx skills add vercel-labs/agent-skills --skill web-design-guidelines -y
npx skills add vercel-labs/agent-skills --skill vercel-react-best-practices -y
npx skills add waynesutton/convexskills --skill convex -y
npx skills add get-convex/agent-skills --skill convex-helpers-guide -y
npx skills add workos/skills --skill workos -y
npx skills add shadcn/ui --skill shadcn -y
```

These give the agent knowledge about:
- **frontend-design** — production-grade UI that avoids generic "AI slop" aesthetics
- **web-design-guidelines** — web design fundamentals and accessibility
- **vercel-react-best-practices** — React and Next.js patterns, RSC, data fetching
- **convex** — Convex schema, queries, mutations, and real-time patterns
- **convex-helpers-guide** — Convex helper utilities and advanced patterns
- **workos** — WorkOS AuthKit integration patterns
- **shadcn** — shadcn/ui component usage and theming

## Step 5: Set up shadcn/ui

Initialize shadcn with defaults (non-interactive):

```bash
npx shadcn@latest init -d -y
```

Then **stop and ask the user:**

> "Time to pick your theme. Open https://ui.shadcn.com/themes in your browser, choose a color palette you like, and give me either:
> - The CLI command it gives you, or
> - The CSS variables to paste into your styles
>
> I'll apply it for you."

Wait for the user to provide the theme command or CSS. Then:

- If they give a CLI command: run it
- If they give CSS variables: paste them into `app/globals.css` (or wherever the project's global styles live), replacing the existing `:root` and `.dark` color variable blocks

## Step 6: Install shadcn components

Run this command (non-interactive):

```bash
npx shadcn@latest add button card input textarea badge dialog -y -o
```

## Step 7: Create agent.md

Create a file called `agent.md` at the project root with this exact content:

```markdown
# Feedback Board — Agent Guide

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

## Installed skills

This project has the following agent skills installed locally in `.claude/skills/`:

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

## Step 8: Verify and hand off

Run the validation command first:

```bash
npx convex dev --once
```

If it passes, start the dev server:

```bash
npm run dev
```

Confirm:
- `localhost:3000` loads in the browser
- No errors in the terminal
- The theme colors are visible on the page

Then tell the user:

> "The project is set up and running. Your code is already on GitHub. Before we start building features, you'll need to sign into Vercel for when it's time to deploy. Run this when you're ready:
>
> ```bash
> npx vercel login
> ```
>
> You don't need to do this now — we'll need it at the end. For now, you're ready to start building."
