---
name: deploy-project
description: Set up continuous deployment for a Next.js + Convex project using Vercel and GitHub. After this one-time setup, every push to the main branch auto-deploys both the frontend (Vercel) and backend (Convex).
---

# Deploy Project

This skill sets up continuous deployment so that every push to GitHub automatically deploys the full application — frontend on Vercel, backend on Convex. After running this skill once, you never need to deploy manually again.

## What this skill does

1. Connects the Vercel project to the GitHub repository
2. Configures the build command so Convex deploys automatically with every Vercel build
3. Creates the Convex deploy key and sets all environment variables — automatically, without exposing secrets
4. Sets up WorkOS production authentication
5. Pushes to GitHub to trigger the first deploy
6. Verifies the live site works

## Prerequisites

Before running this skill:
- The project must be working locally (dev server runs, features work)
- The code must already be on GitHub (the `setup-project` skill handles this)
- `gh` must be authenticated (also handled by `setup-project`)

## CRITICAL: Credential Safety Rules

**NEVER read or print the user's credentials.** This means:
- **NEVER** read `~/.convex/config.json` and display its contents
- **NEVER** read `.env.local` and display secret values
- **NEVER** store credentials in shell variables that appear in command output
- **ALWAYS** use subshell expansion `$(...)` or pipe chains so values flow through commands without appearing in agent output
- When the user must provide a value manually, tell them to run the command themselves in their terminal

## Step 1: Authenticate with Vercel

Check if the user is already authenticated with Vercel:

```bash
npx -y vercel whoami
```

If not authenticated, tell the user:

> "We need to connect to Vercel — this is what puts your site on the internet. This will open your browser to sign in."

Then run:

```bash
npx -y vercel login
```

This is interactive — it opens the browser. Wait for the user to complete it.

Show them where to sign up if needed: "Sign up at vercel.com with your GitHub account — it's free."

## Step 2: Link the Vercel project and connect GitHub

Link the current directory to a Vercel project:

```bash
npx -y vercel link --yes
```

Then connect the GitHub repo so Vercel auto-deploys on push. **You must pass the full GitHub URL** — the shorthand `owner/repo` format will fail:

```bash
GITHUB_URL=$(git remote get-url origin) && npx -y vercel git connect "$GITHUB_URL"
```

## Step 3: Configure the build command

The key to continuous deployment is overriding Vercel's build command so that Convex functions deploy automatically alongside the frontend.

**Create a `vercel.json` file** in the project root (or update it if it already exists):

```json
{
  "buildCommand": "npx convex deploy --cmd 'npm run build' --cmd-url-env-var-name NEXT_PUBLIC_CONVEX_URL"
}
```

**Do NOT use the `VERCEL_BUILD_COMMAND` environment variable** — it does not work reliably. The `vercel.json` approach is the correct way.

**How this works:** When Vercel builds, `npx convex deploy` reads the deploy key, pushes your Convex functions to production, then runs `npm run build` with the production Convex URL injected via `--cmd-url-env-var-name`. Both frontend and backend deploy in one step.

## Step 4: Set environment variables

This step sets all the production environment variables automatically. **No dashboard visits required.** The agent must never see or print any credential values.

### 4a. Convex deploy key (fully automated)

Create a deploy key using the Convex Management API and pipe it directly into Vercel. This single command chain does everything — the key never appears in output:

```bash
DEV_DEPLOYMENT=$(grep "^CONVEX_DEPLOYMENT=" .env.local | sed 's/^CONVEX_DEPLOYMENT=//' | awk '{print $1}' | sed 's/^dev://') && \
PROJECT_ID=$(curl -s -X GET "https://api.convex.dev/v1/deployments/$DEV_DEPLOYMENT" \
  -H "Authorization: Bearer $(python3 -c "import json; print(json.load(open('$HOME/.convex/config.json'))['accessToken'])")" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['projectId'])") && \
PROD_DEPLOYMENT=$(curl -s "https://api.convex.dev/v1/projects/$PROJECT_ID/list_deployments" \
  -H "Authorization: Bearer $(python3 -c "import json; print(json.load(open('$HOME/.convex/config.json'))['accessToken'])")" \
  | python3 -c "import sys,json; deps=json.load(sys.stdin); print(next(d['name'] for d in deps if d['deploymentType']=='prod'))") && \
DEPLOY_KEY=$(curl -s -X POST "https://api.convex.dev/v1/deployments/$PROD_DEPLOYMENT/create_deploy_key" \
  -H "Authorization: Bearer $(python3 -c "import json; print(json.load(open('$HOME/.convex/config.json'))['accessToken'])")" \
  -H "Content-Type: application/json" \
  -d '{"name":"vercel-prod"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['deployKey'])") && \
if [ -n "$DEPLOY_KEY" ]; then \
  printf '%s' "$DEPLOY_KEY" | npx -y vercel env add CONVEX_DEPLOY_KEY production 2>&1; \
  echo "✅ Deploy key created and set on Vercel"; \
else \
  echo "❌ Failed to create deploy key"; \
fi
```

If the pipe-to-stdin approach fails (some Vercel CLI versions require `--value`), use:

```bash
npx -y vercel env add CONVEX_DEPLOY_KEY production --value "$DEPLOY_KEY" 2>&1
```

**If the Management API fails** (e.g., no Convex auth token), fall back to asking the user:

> "I need you to grab a deploy key from Convex. Let me open the page for you."

Open the dashboard — construct the URL from CLI data:
1. Get the dev deployment name from `.env.local` (grep only the `CONVEX_DEPLOYMENT` line)
2. Get deployment info from `GET /v1/deployments/{name}` to find the project
3. The dashboard URL pattern is: `https://dashboard.convex.dev/d/{prod-deployment-name}/settings/deploy-keys`

Then tell the user to run this command themselves:

```
npx -y vercel env add CONVEX_DEPLOY_KEY production --value "PASTE_YOUR_KEY_HERE"
```

### 4b. Convex production URL (fully automated)

The production Convex URL is not a secret. Extract it from the deployment list API response (the `deploymentUrl` field from step 4a) and set it:

```bash
npx -y vercel env add NEXT_PUBLIC_CONVEX_URL production --value "https://$PROD_DEPLOYMENT.convex.cloud"
npx -y vercel env add NEXT_PUBLIC_CONVEX_URL preview --value "https://$PROD_DEPLOYMENT.convex.cloud"
```

**Note:** If the Convex deployment is in a non-default region (e.g., EU), the URL format may include the region: `https://{name}.{region}.convex.cloud`. Check the `deploymentUrl` field from the API response to get the exact URL.

### 4c. WorkOS credentials — test mode (fully automated)

Convex auto-provisions a WorkOS environment when deploying with AuthKit. The credentials are stored as Convex environment variables. Pipe them to Vercel without exposing them:

```bash
npx -y vercel env add WORKOS_CLIENT_ID production \
  --value "$(npx convex env get WORKOS_CLIENT_ID --prod 2>/dev/null | tr -d '\n')"

npx -y vercel env add WORKOS_API_KEY production \
  --value "$(npx convex env get WORKOS_API_KEY --prod 2>/dev/null | tr -d '\n')"
```

Generate and set a cookie encryption password:

```bash
npx -y vercel env add WORKOS_COOKIE_PASSWORD production \
  --value "$(openssl rand -base64 48 | tr -d '/+=' | head -c 64)"
```

**About test credentials:** These auto-provisioned credentials use WorkOS sandbox mode (`sk_test_` prefix). This is **completely fine for launching your app.** Real users can sign up and sign in. The only difference from production keys is enterprise features like custom SAML SSO. You can upgrade to production keys later without losing user data (see Step 7).

### 4d. WorkOS redirect URI

Set the redirect URI after the initial deploy creates the Vercel URL. This happens in Step 6 after the first deployment.

## Step 5: Initial deploy to establish the Vercel URL

**Important:** An initial deploy must happen BEFORE the full git-push pipeline works, because Convex's AuthKit configuration needs the `VERCEL_PROJECT_PRODUCTION_URL` which Vercel only provides during its own builds.

First, commit the `vercel.json` and any pending changes:

```bash
git add vercel.json
git commit -m "Configure production deployment"
```

Run the initial production deploy:

```bash
npx -y vercel --prod --yes
```

Wait for it to complete:

```bash
PROD_URL=$(npx -y vercel ls --prod 2>&1 | tail -1)
npx -y vercel inspect "$PROD_URL" --wait --timeout 180000
```

Tell the user:

> "Your site is being built and deployed. This takes a couple of minutes."

## Step 6: Set redirect URI and trigger full deploy

After the initial deploy completes, get the stable production URL:

```bash
PROD_URL=$(npx -y vercel ls --prod 2>&1 | tail -1)
STABLE_URL=$(npx -y vercel inspect "$PROD_URL" 2>&1 | grep "╶ https://" | grep -v "git-main" | head -1 | awk '{print $2}')
echo "Production URL: $STABLE_URL"
```

Set the WorkOS redirect URI using this URL:

```bash
npx -y vercel env add NEXT_PUBLIC_WORKOS_REDIRECT_URI production \
  --value "${STABLE_URL}/callback"
```

Now push to GitHub to trigger the full deployment pipeline (Convex + Next.js together):

```bash
git push origin main
```

This push triggers Vercel, which runs `npx convex deploy` (deploying the backend with AuthKit configuration) and then `npm run build` (building the frontend). The `convex.json` authKit config uses `${buildEnv.VERCEL_PROJECT_PRODUCTION_URL}` to automatically configure WorkOS redirect URIs and CORS origins.

Monitor the deployment:

```bash
DEPLOY_URL=$(npx -y vercel ls --prod 2>&1 | tail -1)
npx -y vercel inspect "$DEPLOY_URL" --wait --timeout 180000
```

## Step 7: Verify the live site

After the deployment completes, get the live URL and test it:

```bash
PROD_URL=$(npx -y vercel ls --prod 2>&1 | tail -1)
STABLE_URL=$(npx -y vercel inspect "$PROD_URL" 2>&1 | grep "╶ https://" | grep -v "git-main" | head -1 | awk '{print $2}')
```

Tell the user:

> "Your app is live at [URL]. Open it in your browser and check that:
> 1. The page loads and looks right
> 2. You can sign in (click the sign-in button)
> 3. The features work — try submitting something, voting, etc.
>
> If everything works, you're live! From now on, every time you push code to GitHub, both your site and your backend update automatically."

If the user reports an issue, go to the Recovery section.

## Step 8: Upgrade to WorkOS production keys (optional, one dashboard visit)

Tell the user:

> "Your app is using WorkOS test credentials right now. This is totally fine — real users can sign up and everything works. But when you're ready, you can upgrade to production credentials for enterprise features.
>
> This is the one step that requires visiting a dashboard. Here's what to do:"

**Guide the user through these steps:**

1. Go to [dashboard.workos.com](https://dashboard.workos.com)
2. Switch to the **Production** environment using the dropdown at the top
3. If production isn't available, complete WorkOS's "Unlock Production" checklist
4. Go to **API Keys** and generate a production API key (`sk_live_...`)
5. Copy both the **API Key** and the **Client ID** for the production environment

**Important:** Production API keys can only be viewed **once** at creation time. Save it immediately.

Then tell the user to run these commands in their own terminal (the agent must NOT run these — the user pastes their own values):

```
npx -y vercel env rm WORKOS_CLIENT_ID production --yes
npx -y vercel env rm WORKOS_API_KEY production --yes
npx -y vercel env add WORKOS_CLIENT_ID production --value "PASTE_YOUR_PROD_CLIENT_ID"
npx -y vercel env add WORKOS_API_KEY production --value "PASTE_YOUR_PROD_API_KEY"
npx convex env set WORKOS_CLIENT_ID "PASTE_YOUR_PROD_CLIENT_ID" --prod
npx convex env set WORKOS_API_KEY "PASTE_YOUR_PROD_API_KEY" --prod
```

After the user confirms they've run the commands, redeploy:

```bash
npx -y vercel --prod --yes
```

**About the migration:** When switching from test to production WorkOS keys, existing users from the test environment do NOT carry over — they are in separate WorkOS environments. For an MVP (under ~1,000 users), this is usually fine. If the user has significant existing users they need to migrate, WorkOS provides a User Import API that can recreate users in the production environment. This is a separate, more advanced task.

## Recovery

Common issues and how to fix them:

- **"Invalid client ID" error on sign-in:** This usually means a trailing newline character got into the environment variable. Remove and re-add it:
  ```bash
  npx -y vercel env rm WORKOS_CLIENT_ID production --yes
  npx -y vercel env add WORKOS_CLIENT_ID production \
    --value "$(npx convex env get WORKOS_CLIENT_ID --prod 2>/dev/null | tr -d '\n')"
  npx -y vercel --prod --yes
  ```

- **Convex deploy fails with "VERCEL_PROJECT_PRODUCTION_URL not set":** This means the initial Vercel deploy hasn't happened yet. Run `npx -y vercel --prod --yes` first to establish the Vercel project, then push to GitHub.

- **500 error on live site:** Usually means WorkOS environment variables are missing from Vercel. Check with `npx -y vercel env ls` and verify all five are set: `CONVEX_DEPLOY_KEY`, `NEXT_PUBLIC_CONVEX_URL`, `WORKOS_CLIENT_ID`, `WORKOS_API_KEY`, `WORKOS_COOKIE_PASSWORD`.

- **Build fails on Vercel:** Check the build logs: `npx -y vercel inspect <deployment-url> --logs`. The error is usually a missing environment variable or a code error. Fix it, push again.

- **`vercel git connect` fails with "Failed to parse URL":** You must pass the full GitHub URL, not the `owner/repo` shorthand:
  ```bash
  npx -y vercel git connect "$(git remote get-url origin)"
  ```

- **`vercel env add` says "already exists":** Remove first, then re-add:
  ```bash
  npx -y vercel env rm VARIABLE_NAME production --yes
  npx -y vercel env add VARIABLE_NAME production --value "new-value"
  ```

- **Data doesn't show on live site:** The production Convex deployment is empty (separate from your dev database). Seed data through the Convex dashboard or run a seed function against production.

## What happens after this

Once this skill completes, the deployment pipeline is fully automatic:

1. You make changes locally
2. You push to GitHub (`git push`)
3. Vercel detects the push and starts a build
4. The build command runs `npx convex deploy`, which deploys your Convex functions and configures WorkOS
5. Then `npm run build` runs, building the frontend with the production Convex URL
6. Vercel publishes the new frontend
