# End-to-End CI/CD with GitHub Actions: React App → GitHub Pages + SonarCloud Quality Gate

This project shows how to take a simple React app and put **real CI/CD** around it using **GitHub Actions**, **GitHub Pages**, and **SonarCloud**.

By the end, your repository can:

- ✅ Test code on every change
- ✅ Build a production bundle
- ✅ Cache dependencies to speed up CI
- ✅ Package build output as artifacts
- ✅ Deploy automatically to GitHub Pages
- ✅ Run SonarCloud on every Pull Request as a quality gate

All of this is fully automated from your GitHub repo.

---

## 1. Big Picture – What We’re Actually Doing

Think of the repo as a **factory**:

1. Developer pushes code / opens a Pull Request
2. GitHub Actions (the factory robot) automatically:
    - Runs tests
    - Builds the React app
    - Packages build output (artifact)
    - Deploys the app to **GitHub Pages**
    - For PRs: asks **SonarCloud** to inspect the code quality before merge

No manual “run tests locally”, “build locally”, “FTP files somewhere”.

You push → the pipeline does the rest.

---

## 2. Core Concepts (Explained Simply)

### 2.1 GitHub Actions

- **GitHub Actions** is an **automation service inside GitHub**.
- It listens to events like:
    - `push` (code pushed to a branch)
    - `pull_request` (PR opened/updated)
- When an event happens, it runs a **workflow** (a YAML file under `.github/workflows/`).

Key terms:

- **Workflow** → the full automation recipe (YAML file).
- **Job** → a block of work (e.g. `test`, `build`, `deploy`).
- **Step** → one task in a job (`run:` or `uses:`).
- **Runner** → temporary machine in the cloud where steps execute (e.g. `ubuntu-latest`).

---

### 2.2 CI vs CD

- **Continuous Integration (CI)** = every change:
    - ✅ Run tests
    - ✅ Check quality/security
    - ✅ Build the project
    - ✅ Produce an **artifact** (final build output)
- **Continuous Deployment (CD)** = once artifact is ready:
    - 🚀 Automatically deploy it to an environment (here: **GitHub Pages**)

Analogy:

Factory line → **test → assemble → pack → ship**

- CI = test + assemble + pack
- CD = ship

---

### 2.3 Runner Images

- A **runner** is a fresh VM created for each workflow run.
- `runs-on: ubuntu-latest` means:
    - “Give me a Linux (Ubuntu) machine with common tools pre-installed.”
- Other options: `windows-latest`, `macos-latest`, or self-hosted runners.

You never log into it; GitHub Actions runs commands on it for you.

---

### 2.4 Artifacts

- **Artifact** = any file/folder produced during the workflow that you want to keep.
- In this project:
    - `npm run build` creates `dist/` → this is our **artifact** (production-ready HTML/CSS/JS).
- We:
    - **Upload** this artifact after build
    - **Download** it in the deploy job for deployment

Analogy:

Bake cake in kitchen A (build job) → box it (artifact) → delivery team (deploy job) picks up the box instead of re-baking.

---

### 2.5 Caching

- Installing dependencies (`npm install`) on every job is slow.
- We use **`actions/cache`** to store `node_modules/` and reuse it:
    - First run → no cache → install & cache
    - Next runs (no dependency changes) → re-use cache, skip install
- Cache key uses `hashFiles('package-lock.json')`:
    - Change dependencies → `package-lock.json` changes → new cache → fresh install

---

### 2.6 Filters (Branches, Paths, Manual Trigger, Skip CI)

We control when workflows run:

- Only for certain **branches**:
    
    ```yaml
    on:
      push:
        branches:
          - main
          - feature/*
    
    ```
    
- Ignore certain **paths**:
    
    ```yaml
        paths-ignore:
          - README.md
          - .github/workflows/**
    
    ```
    
    → Don’t run the whole pipeline if only docs/workflow files changed.
    
- Allow **manual runs**:
    
    ```yaml
    workflow_dispatch: {}
    
    ```
    
- Skip CI with commit message:
    
    ```
    [skip ci]  or  [ci skip]
    
    ```
    

---

### 2.7 Composite Actions (Custom Reusable Logic)

We created our own mini-action:

- Path: `.github/actions/setup-node/action.yml`
- Purpose:
    - Setup Node.js
    - Cache dependencies
    - Install dependencies

Instead of duplicating those steps in each job, we call:

```yaml
- name: Setup Node and install dependencies
  uses: ./.github/actions/setup-node
  with:
    node-version: 20    # or omit to use default

```

This keeps the workflow **DRY** (Don’t Repeat Yourself).

---

### 2.8 GitHub Pages

- **GitHub Pages** is free hosting for **static sites** (HTML/CSS/JS).
- We deploy the built React app (`dist/` folder) to GitHub Pages via Actions.
- Once deployed, the app is available at:
    
    ```
    https://<username>.github.io/<repo-name>
    
    ```
    

We use:

- `actions/upload-pages-artifact` → to package `dist/`
- `actions/deploy-pages` → to publish to GitHub Pages

---

### 2.9 SonarCloud

- **SonarCloud** scans code for:
    - Bugs
    - Code smells
    - Security issues
    - Duplications
- We integrated SonarCloud with GitHub so that:
    - Every **Pull Request** to `main` runs a **quality scan**
    - It posts results on the PR (no new issues / issues found, etc.)

This acts as a **quality gate**:

You don’t merge dirty code into `main` without seeing the analysis.

---

## 3. Step-By-Step: What We Actually Implemented

### Step 1 – Setup React Project and Push to GitHub

- Created a simple React project.
- Initialized Git and pushed code to GitHub:
    - `git init`
    - `git remote add origin <repo-url>`
    - `git push -u origin main`

**Result:** Code is on GitHub → ready for automation.

---

### Step 2 – First Workflow: “Hello World” on GitHub Actions

File: `.github/workflows/hello.yml`

- Triggered on every `push`.
- Single job `greet` running on `ubuntu-latest`.
- Steps:
    - `echo "Hello world"`
    - `echo "Bye bye"`

**Concepts learned:**

- What a **workflow**, **job**, **step**, and **runner** are.
- How YAML indentation and key/value structure works.
- How to view logs in **Actions** tab.

---

### Step 3 – Add Test, Build, and Deploy Jobs with Dependencies

File: `.github/workflows/deploy-dist.yml` (example name)

Jobs:

1. **`test` job**
    - `npm test`
2. **`build` job**
    - `needs: test` → runs only if tests pass
    - `npm run build` → produces `dist/` folder
3. **`deploy` job**
    - `needs: build`
    - For now, just prints `Deployment completed` and lists `dist` content

**Key patterns:**

- Jobs run in **sequence** using `needs`.
- If **test fails**, `build` and `deploy` are skipped.

---

### Step 4 – Speed Up CI with Caching (`actions/cache`)

We added caching around `npm install`:

```yaml
- name: Cache dependencies
  id: cache
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm install

```

Applied to both:

- `test` job
- `build` job

**Effect:**

- First run: `npm install` runs once per job.
- Next runs (no dependency changes): `npm install` is skipped → faster pipeline.

---

### Step 5 – Use Artifacts to Pass Build Output Between Jobs

In `build` job:

```yaml
- name: Upload dist artifact
  uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: dist

```

In `deploy` job:

```yaml
- name: Download dist artifact
  uses: actions/download-artifact@v4
  with:
    name: dist-files
    path: dist

- name: Show files being deployed
  run: |
    echo "Deploying these files:"
    ls dist

```

**What this enabled:**

- Build **once** per workflow.
- Reuse the same `dist/` output in the deploy job.
- Download artifacts manually from the Actions UI if needed (for debugging).

---

### Step 6 – Add Filters (Branches, Paths) and Manual Trigger

In the deploy workflow, we refined `on:`:

```yaml
on:
  push:
    branches:
      - main
      - feature/*
    paths-ignore:
      - README.md
      - .github/workflows/**
  workflow_dispatch: {}

```

Behavior:

- Workflow runs only if:
    - Pushed to `main` or a branch starting with `feature/`
- It **ignores**:
    - Changes only in `README.md`
    - Changes only to workflow files themselves
- We can also **start the workflow manually** from the Actions UI.

We also used commit message tags like `[skip ci]` to skip runs when making trivial changes.

---

### Step 7 – Custom Composite Action: `setup-node`

Folder: `.github/actions/setup-node/action.yml`

This composite action:

- Sets up Node.js
- Caches `node_modules`
- Runs `npm install` only if cache is missing
- Accepts a `node-version` input

Example:

```yaml
name: Setup Node and install dependencies
description: Setup Node.js, cache dependencies, and install them

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '22'

runs:
  using: composite
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

    - name: Install dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      shell: bash
      run: npm install

```

Usage in any job:

```yaml
- uses: actions/checkout@v4

- name: Setup Node and install dependencies
  uses: ./.github/actions/setup-node
  with:
    node-version: 20   # optional; if omitted → default 22

```

**What we achieved:**

- Removed duplicated setup from `test` and `build`.
- Centralized Node setup logic.
- Allowed different Node versions per job.

---

### Step 8 – Deploy the React App to GitHub Pages via CI/CD

New workflow: `.github/workflows/deploy-react-to-pages.yml`

**Build job:**

- Checkout code
- Use `setup-node` composite action
- `npm run build` → generate `dist/`
- Use Pages-specific artifact upload:

```yaml
- name: Upload GitHub Pages artifact
  uses: actions/upload-pages-artifact@v4
  with:
    name: github-pages
    path: dist

```

**Deploy job:**

```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest
  permissions:
    pages: write
    id-token: write
  environment:
    name: github-pages
  steps:
    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
      with:
        token: ${{ secrets.GITHUB_TOKEN }}

```

Additional changes:

- In `package.json`:
    
    ```json
    "homepage": "https://<username>.github.io/<repo-name>"
    
    ```
    
- Repo Settings → Pages:
    - Source set to **GitHub Actions**
    - Environment `github-pages` configured (e.g. main only)

**Outcome:**

- Push to `main` → CI builds and deploys to GitHub Pages.
- App live at:
    
    ```
    https://<username>.github.io/<repo-name>
    
    ```
    

---

### Step 9 – Add SonarCloud Quality Gate on Pull Requests

We integrated **SonarCloud** to analyze code on PRs.

**Setup steps:**

1. Log in to SonarCloud with GitHub.
2. Create an organization and import this repo.
3. Set **analysis method** to **GitHub Actions**.
4. Copy generated Sonar GitHub Actions YAML and place in:
    - `.github/workflows/sonar-cloud.yml`
5. Create `sonar-project.properties` in repo root with project details.
6. Generate a **Sonar token** and store it in GitHub:
    - Settings → Secrets → Actions → New secret `SONAR_TOKEN`
7. Reference this in the workflow:
    
    ```yaml
    env:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    
    ```
    

**Workflow behavior:**

- Triggered on:
    - `push` to `main`
    - `pull_request` events: `opened`, `reopened`, `synchronize`
- Steps:
    - Checkout code
    - Run SonarCloud scanner
- SonarCloud posts results on the PR:
    - Issues found / none
    - Quality gate status

**What we gained:**

- Every PR to `main` is analyzed **before merge**.
- You see whether new code introduces:
    - Bugs
    - Code smells
    - Security hotspots
- You can enforce “no merge if quality gate fails”.

---

## 4. What We Achieved Overall

By the end of this setup, this repo demonstrates:

- ✅ **Full CI/CD pipeline for a React app**
    - Test → Build → Package → Deploy
- ✅ **GitHub Actions fundamentals**
    - Workflows, jobs, steps, runners
    - Events (`push`, `pull_request`, `workflow_dispatch`)
- ✅ **Performance optimizations**
    - Caching Node dependencies
    - Avoiding unnecessary runs with branch/path filters and `[skip ci]`
- ✅ **Clean architecture for pipelines**
    - Separate jobs for test, build, deploy
    - Job dependencies with `needs`
- ✅ **Reusability**
    - Custom composite action (`setup-node`) to avoid duplicated YAML
- ✅ **Static hosting**
    - Automated deployment to **GitHub Pages**
- ✅ **Code quality & governance**
    - SonarCloud integration as a PR quality gate
    - Secrets management via GitHub repository secrets

---

Happy Learning!