# CI/CD Crash Course: Commands and Code

## Clone the starter app

The starter includes the Express/TypeScript app, a health endpoint test, TypeScript configuration, and the basic npm scripts.

Run these commands in Bash or Zsh (Git Bash on Windows).

```bash
git clone https://github.com/bradtraversy/cicd-lab-starter.git cicd-lab
cd cicd-lab
code .
```

From inside the newly cloned `cicd-lab` folder, remove the starter's Git history and remote connection, then initialize your own repository on `main`. Your app files are preserved.

```bash
rm -rf .git
git init -b main
```

The starter already declares Node.js 24 in `engines` and includes matching Node types. Check your local version before installing dependencies:

```bash
node --version
```

If the output does not start with `v24.`, install or switch to Node.js 24 before continuing. The `engines` field does not switch your local Node version.

Install dependencies from the lockfile:

```bash
npm ci
```

## Run the app

```bash
npm run dev
```

Visit `http://localhost:3000` and `http://localhost:3000/api/health`. The health endpoint should return HTTP 200 with this JSON:

```json
{ "status": "ok" }
```

## Run the existing checks

The starter includes these scripts in `package.json`:

```json
"scripts": {
  "dev": "tsx watch src/server.ts",
  "typecheck": "tsc --noEmit",
  "test": "node --import tsx --test test/app.test.ts",
  "build": "tsc",
  "start": "node dist/server.js"
}
```

The test in `test/app.test.ts` starts its own server and checks that `/api/health` returns HTTP 200 and `{ "status": "ok" }`.

Run the checks in a second terminal. The build compiles JavaScript into `dist`.

```bash
npm test
npm run typecheck
npm run build
```

## Add one verification command

Add this entry inside the `scripts` object in `package.json`, including the necessary comma between entries:

```json
"verify": "npm run typecheck && npm test && npm run build"
```

Run all checks with one command. If a check fails, `&&` stops the remaining commands.

```bash
npm run verify
```

## Commit the starting app and connect GitHub

The starter's `.gitignore` excludes dependencies, build output, and environment secrets. Include `package-lock.json` in your commit.

```bash
git status --short
git add .
git commit -m "feat: add starter app with local verification"
```

Create an empty `cicd-lab` repository on GitHub without a README, license, or gitignore. Use a public repository to use rulesets on GitHub Free.

```bash
git remote add origin https://github.com/YOUR_USERNAME/cicd-lab.git
git push -u origin main
```

## Deploy the small app on Render

Connect your GitHub account to Render and create a Node Web Service from your own repository, branch `main`, with a blank root directory and the Free instance for this demo.

Build Command:

```bash
npm ci && npm run build
```

Start Command:

```bash
npm start
```

```text
Health Check Path: /api/health
Auto-Deploy: On Commit
```

Wait for a successful deployment, then confirm the public home page and `/api/health` response.

## Add GitHub Actions

```bash
git switch -c feature/github-actions-ci
```

```bash
mkdir -p .github/workflows
```

Create `.github/workflows/ci.yml`; the job runs for pull requests into `main` and pushes to `main`.

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  verify:
    name: Verify
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Check out repository
        uses: actions/checkout@v7

      - name: Set up Node.js
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Verify application
        run: npm run verify
```

Stop the development server, then reproduce the CI installation and checks locally.

```bash
npm ci
npm run verify
```

Review, commit, and publish this branch.

```bash
git add .
git commit -m "ci: add GitHub Actions verification workflow"
git push -u origin feature/github-actions-ci
```

Open a pull request into `main` on GitHub.

Wait for Verify to pass, inspect its Actions logs, then merge; merging triggers another run for `main`.

Merge the pull request using **Create a merge commit**, click **Delete branch** on GitHub if it is still available, then run:

```bash
git switch main
git pull --ff-only
git branch -d feature/github-actions-ci
```

## Deploy only after CI passes

After the GitHub Actions workflow is merged, wait for the Verify run on `main` to pass. In the Render web service settings, change Auto-Deploy from On Commit to After CI Checks Pass. Future pushes to `main` will deploy only after their CI checks pass.

## Require Verify before merging

In GitHub Settings > Rules > Rulesets, create `Protect main` with these settings; the workflow reports results and the ruleset enforces them.

```text
Enforcement: Active
Target: Default branch (main)
Require a pull request before merging: Enabled
Required approvals: 0 for this solo demo
Require status checks to pass: Enabled
Required check: Verify (GitHub Actions)
Bypass list: Empty
```

## Send a normal change through the pipeline

```bash
git switch -c feature/update-homepage
```

Add to the home page HTML in `src/app.ts`.

```html
<p>Verified by GitHub Actions. Deployed on Render.</p>
```

Run the local checks.

```bash
npm run verify
```

Review, commit, and publish this branch.

```bash
git add .
git commit -m "feat: update homepage deployment message"
git push -u origin feature/update-homepage
```

Open a pull request into `main` on GitHub.

Wait for Verify before merging.

Merge the pull request using **Create a merge commit**, click **Delete branch** on GitHub if it is still available, then run:

```bash
git switch main
git pull --ff-only
git branch -d feature/update-homepage
```

Wait for the `main` check and Render deployment, then confirm the new message on the public page.

## Demonstrate a failing test

```bash
git switch -c feature/version-endpoint
```

Append to `test/app.test.ts` before creating the route.

```ts
test('GET /api/version returns the current version', async () => {
  const response = await fetch(`${baseUrl}/api/version`);

  assert.equal(response.status, 200);
  assert.deepEqual(await response.json(), {
    version: process.env.APP_VERSION ?? 'development',
  });
});
```

This deliberately fails with HTTP 404 instead of 200.

```bash
npm test
```

Review, commit, and publish this branch.

```bash
git add .
git commit -m "test: demonstrate missing version endpoint"
git push -u origin feature/version-endpoint
```

Open a pull request into `main` on GitHub.

Inspect the failed Verify job and blocked merge; leave this pull request open.

Fix the failure by adding this route before `export default app` in `src/app.ts`.

```ts
app.get('/api/version', (_request, response) => {
  response.json({
    version: process.env.APP_VERSION ?? 'development',
  });
});
```

Confirm both tests and the build pass.

```bash
npm run verify
```

```bash
git add .
git commit -m "feat: add version endpoint"
git push
```

The existing pull request updates automatically; wait for its new Verify run to pass.

Merge the pull request using **Create a merge commit**, click **Delete branch** on GitHub if it is still available, then run:

```bash
git switch main
git pull --ff-only
git branch -d feature/version-endpoint
```

After deployment, open `/api/version`; with no APP_VERSION set, expect:

```json
{ "version": "development" }
```

## Preview the small app

A preview deployment is a temporary copy of your app built from a pull request, with its own URL, so you can check the change in a browser before merging. Production keeps serving `main`, and Render deletes the preview when the pull request is merged or closed.

In the Render Dashboard, open your web service's **Previews** tab, set **Pull Request Previews** to **Automatic**, and save. Each preview is a separate running instance, so check the pricing shown. To preview only selected pull requests, choose **Manual** and put `[render preview]` in the pull request title. See [Render service previews](https://render.com/docs/service-previews).

```bash
git switch -c feature/preview-demo
```

Add to the home page HTML in `src/app.ts`.

```html
<p>Preview deployment</p>
```

Verify the change.

```bash
npm run verify
```

Review, commit, and publish this branch.

```bash
git add .
git commit -m "feat: add preview deployment message"
git push -u origin feature/preview-demo
```

Open a pull request into `main` on GitHub and leave it unmerged. With Automatic enabled, use a normal title:

```text
Add preview deployment message
```

If you selected Manual instead, use this title to request a preview:

```text
[render preview] Add preview deployment message
```

Opening the pull request triggers the preview; you do not need to merge it first. Wait for Verify and the preview deployment. Open the preview from the pull request's **View deployment** link or the service's **Previews** tab in Render.

Compare the two URLs while the pull request is still open: the preview should display the new message, and production should not. Only merge after reviewing the preview.

Merge the pull request using **Create a merge commit**, click **Delete branch** on GitHub if it is still available, then run:

```bash
git switch main
git pull --ff-only
git branch -d feature/preview-demo
```

Confirm the production message after deployment and that Render removes the temporary preview.
