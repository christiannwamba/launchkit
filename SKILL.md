---
name: video-recording-runbook
description: Build a production runbook from a video outline. Use when a user has an outline for a demo, tutorial, coding video, or prompt-based AI workflow video and wants a concrete prep plan, scene-by-scene run sequence, pre-record tasks, and reset guidance without fluff.
---

# Video Recording Runbook

Use this skill when the user already has a video outline and wants an operational document for recording it.

This skill is not limited to coding videos. It also applies to prompt-based AI workflows, operator demos, tool walkthroughs, and other recorded step-by-step workflows.

The output is not a flat checklist by default. The default output is a runbook.

## Input

Start from the user's outline or from a file such as `docs/outline.md`.

Treat the outline as the narrative source of truth:

- what the video is trying to teach
- what the demo is supposed to do
- what tools or services appear in the story
- what needs to happen on screen

## Output

Default output:

- one production-ready `recording-runbook.md`

The runbook may be built incrementally over multiple turns. If the user wants to work section by section, update only the requested section instead of generating the whole document again.

Default collaboration pattern:

1. outline the runbook structure first
2. debate and refine the outline with the user
3. build the document section by section in this order:
   - `Prep`
   - `Run`
   - `Recovery`

## Document model

The runbook should normally use this structure:

- `Prep`
- `Run`
- `Recovery`

Do not force all three sections at once. If the user wants to draft only `Prep` first, do that.

### Prep

`Prep` is only for what must be decided or prepared before the live run starts.

Keep it tight. It should usually contain:

- required accounts
- locked stack or product decisions
- concrete pre-record tasks

Do not put prompts here by default.

Do not put local tool installation steps here by default.

### Run

`Run` is the main body of the runbook.

It must be a literal ordered sequence of scenes, not a bucket of topics.

Each scene should usually contain:

- goal
- what happens on screen
- what must already be true before this scene
- success state before moving on
- recovery if this scene fails

If a prompt is needed, place it inside the relevant scene. Do not create a separate prompt inventory unless the user explicitly asks for one.

If a setup scene needs local tools or skills, follow the "Handle missing dependencies gracefully" principle: check, ask permission, install, continue.

### Recovery

`Recovery` is only for global recovery information.

Do not duplicate scene-level recovery here.

It should usually contain:

- recovery rules
- global reset
- full reset

Do not add fallback assets by default.

## Principles

These are hard rules. Apply them to every runbook.

### Introduce services at point of need

Do not front-load account creation, signup pages, or service onboarding into a prep section. Introduce each service in the scene where it is first used. If a service account is needed to complete a step, that step is where the signup happens — not in a separate "accounts overview" scene.

### Clarify the audience before setting the tone

Before deciding how technical or non-technical the runbook's prompts, commands, and explanations should be, ask the user:

- **Who is the audience?** (beginners, intermediate developers, experienced engineers)
- **What should they do on camera?** (just follow prompts? run their own commands? understand the code?)

If the audience is non-technical:
- Prompts describe product behavior, not implementation details
- The audience should only: (1) paste prompts, (2) create accounts in dashboards, (3) run interactive commands that are unavoidable (browser auth flows)
- Never mention file paths, function names, schema fields, or component names in prompts

If the audience is technical:
- Prompts can reference implementation details when helpful
- The audience may run manual commands, inspect code, or debug directly
- Show more of the system, not just the surface

Do not assume either direction — ask first.

### Prefer non-interactive CLI commands

For every CLI command in the runbook, find the non-interactive version first. Use flags like `-y`, `--yes`, `-d`, `--defaults`, `-t <template>` to skip interactive menus.

The only commands that should be interactive are ones that genuinely cannot be automated — like `gh auth login` (opens a browser) or `npx vercel login` (opens a browser).

### Handle missing dependencies gracefully

If a setup scene discovers a missing tool or dependency, the skill or prompt should:

1. Tell the user what's missing and why it's needed
2. Ask their permission before installing it
3. Continue executing the remaining steps after installation

Do not stop the entire flow to present a list of things to install. Check, ask, install, continue.

### Push to source control early

Do not wait until the deployment scene to push code to GitHub (or equivalent). Push immediately after the project is scaffolded. Every subsequent change builds on a tracked baseline. This also means the deployment scene is simpler — it only needs to push the latest changes, not set up the repo from scratch.

### The skill creates the folder

Do not assume the project folder already exists. The first action in any setup scene should create the project folder and move into it. The user should not need to do this manually before pasting the prompt.

### Plans go in docs/, named by task

Build plans are not generic `plan.md` files at the project root. They go in the `docs/` folder and are named after the task they cover:

- `docs/feedback-board-plan.md`
- `docs/add-payments-plan.md`
- `docs/homepage-redesign-plan.md`

If the user doesn't specify a name, derive one from the task description.

### Product-first planning

When a runbook includes a build plan or step-by-step construction of a product, order steps by product surface — not by technical layer.

Each step should complete something the user can see and verify (a page, a form, a button that works). Do not organize as "set up all backend, then all frontend."

### agent.md is minimal

If the runbook creates an `agent.md` for the project, keep it tight:

- What skills are installed and what they're for
- Key commands to run the project, validate changes, and deploy
- Nothing else — no explanations, no architecture docs, no tutorials

## Workflow

### 1. Clarify the scope first

Before proposing the runbook structure, make sure the scope of the workflow is clear.

If the scope is underspecified, start by clarifying:

- what is being taught or demonstrated
- what the final outcome should be
- which features or steps must be included
- which features or steps are intentionally out of scope
- whether the workflow is coding-heavy, prompt-heavy, operator-heavy, or mixed
- **who the audience is** (non-technical beginners, intermediate, experienced)

Do not jump straight to tools or architecture if the scope is still blurry.

### 2. Read the outline and infer the recording workflow

Figure out:

- the product or demo being built
- the services and accounts involved
- which steps belong in prep versus live execution
- which moments are better handled as pre-record material
- which scenes will need explicit recovery guidance

### 3. Explore before asking

Before asking the user questions, inspect the local repo and any provided outline or docs.

Only ask about decisions that materially change the runbook.

Do not ask the user to restate structural preferences once they have already clarified them.

When the task involves building, setup, or product design, break the problem into parts and explore the likely tool choices for each part before drafting:

- app framework
- styling system
- component system
- auth path
- data or backend path
- deployment path
- AI surface shown on screen
- design skill or best-practice guidance to apply

Do not wait for the user to volunteer these. Surface them proactively.

### 4. Lock only high-impact assumptions

When assumptions are needed, keep them short and concrete.

Examples:

- which AI surface is on screen
- whether the run is mostly live or mostly staged
- whether a given account flow should be shown live or pre-recorded
- whether auth is managed through one stack path or another

Present assumptions in a way the user can verify or override.

For each meaningful subsystem, prefer this pattern:

- what assumption you are making
- why it is the best current default
- what alternative the user might choose instead

Examples:

- `Framework: Next.js App Router unless you want a different frontend stack.`
- `Styling: Tailwind unless you want plain CSS or another design system.`
- `UI system: shadcn unless you want a fully custom component layer.`
- `Auth: WorkOS through Convex AuthKit unless you want a different auth story.`

If a design, architecture, or tool choice is still underspecified after exploration, ask the user to verify it before drafting that part of the runbook.

### 5. Outline before drafting

Before writing the runbook body, propose the document shape first when the structure is still unsettled.

Refine that outline with the user until the section boundaries are clear.

If not already in plan mode, suggest switching to plan mode before drafting the runbook so the structure, assumptions, and section boundaries can be agreed before implementation.

### 6. Write the requested section

If the user asks for only `Prep`, write only `Prep`.

If the user asks for the whole runbook, write the whole runbook.

When building section by section, default to this order:

- `Prep`
- `Run`
- `Recovery`

Prefer updating the existing document over creating duplicates.

## Writing rules

- Do not open with fluff such as `Purpose of this section`, `Narrative target`, or similar framing unless the user explicitly asks for it.
- Prefer concrete operational statements over explanation.
- `Concrete` means the instruction includes the asset needed to complete it whenever one exists:
  - the exact link to open
  - the exact command to run
  - the exact prompt to paste
  - the exact code snippet to paste or replace
  - the exact file or screen to open
- Do not stop at a vague instruction like `record the signup flow`, `set up auth`, or `update the component`.
- If the action depends on a specific external page, provide the page link.
- If the action depends on a command, provide the command.
- If the action depends on a prompt, provide the prompt inline at the point of use.
- If the action depends on editing code, provide the code snippet or clearly identify the file and the exact code to replace.
- If a step says to record something, specify exactly what screen to capture and how the user gets to that screen.
- If a step cannot yet be made concrete without a product or workflow decision, do not guess. Ask a clarifying question and use the answer to produce a concrete instruction.
- Do not create long flat lists when a stronger hierarchy exists.
- If the user wants a scene sequence, write numbered scenes in execution order.
- If the user wants setup instructions embedded in the run, do not move them into prep.
- If the user wants prompts to appear only where used, do not create a master prompt section.
- When describing pre-record work, name the exact flow to capture. Avoid vague phrases like `record any account creation` or `record dashboard setup`.
- If documentation exists for an external service, use it to make pre-record steps concrete.
- Keep the document easy to scan. Remove anything that does not help the user execute the recording.

## File-writing rules

When writing the runbook:

- prefer `docs/recording-runbook.md` if a `docs/` folder exists
- otherwise create a docs folder and write `recording-runbook.md` inside it
- update the existing runbook if one already exists instead of creating duplicates

## Success criteria

The runbook is good if a user can:

1. prepare what must exist before recording without guessing
2. follow the live recording sequence in order
3. recover from scene failures without losing the structure of the run
4. reset or rebuild the workflow when needed
