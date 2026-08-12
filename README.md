# 🚀 GitHub Actions — Multistage Workflow Tutorial

A hands-on learning repository for understanding **GitHub Actions workflows, jobs, steps, dependencies, working directories, runners, conditions, and job isolation**.

This project intentionally uses a simple repository structure so that each GitHub Actions concept can be observed clearly without the complexity of a real CI/CD pipeline.

---

## 📚 What You Will Learn

By completing this tutorial, you will understand:

* 🧩 What a GitHub Actions **workflow** is
* 🏗️ Difference between **workflow, jobs, and steps**
* 🖥️ `runs-on` and GitHub-hosted runners
* 📥 Why `actions/checkout` is required
* 📁 `defaults.run.working-directory`
* 📂 Step-level `working-directory`
* 🔗 `needs` and job dependencies
* 🔀 Multiple dependencies with `needs: [job1, job2]`
* 🛑 Conditional jobs using `if`
* 🌿 Branch-based execution
* 🔄 How jobs execute on separate runners
* 🧠 How the GitHub Actions dependency graph works

---

# 🏗️ Repository Structure

```text
workflow-tutorial/
│
├── .github/
│   └── workflows/
│       └── tutorial.yml
│
├── test_environment/
│   ├── dev/
│   │   └── main.tf
│   │
│   ├── preprod/
│   │   └── main.tf
│   │
│   └── prod/
│       └── main.tf
│
├── test_module/
│   ├── test_resource1/
│   │   └── main.txt
│   │
│   └── test_resource2/
│       └── main.txt
│
├── .gitignore
├── LICENSE
└── README.md
```

The directories are deliberately simple.

The Terraform files and text files are mainly used to demonstrate **which working directory each step is executing in**.

---

# 🔄 Workflow Overview

The workflow contains three jobs:

```text
                  ┌──────────────┐
                  │     SCAN     │
                  │    Ubuntu    │
                  └──────┬───────┘
                         │
                       needs
                         ↓
                  ┌──────────────┐
                  │     PLAN     │
                  │    macOS     │
                  └──────┬───────┘
                         │
                       needs
                         ↓
                  ┌──────────────┐
                  │    APPLY     │
                  │    Ubuntu    │
                  └──────────────┘
```

The workflow therefore follows:

```text
SCAN → PLAN → APPLY
```

However, `APPLY` has an additional condition:

```yaml
if: github.ref == 'refs/heads/main'
```

Therefore:

### Feature branch

```text
Push
 │
 ↓
SCAN ✅
 │
 ↓
PLAN ✅
 │
 ↓
APPLY ⏭️ Skipped
```

### Main branch

```text
Push
 │
 ↓
SCAN ✅
 │
 ↓
PLAN ✅
 │
 ↓
APPLY ✅
```

This represents a common CI/CD pattern:

> **Validate and plan on every push, but apply only from the protected main branch.**

---

# 🧩 Understanding Workflow → Jobs → Steps

A GitHub Actions workflow has a hierarchy.

```text
Workflow
│
├── Job
│   ├── Step
│   ├── Step
│   └── Step
│
├── Job
│   ├── Step
│   └── Step
│
└── Job
    ├── Step
    └── Step
```

In this project:

```text
Multistage Workflow Tutorial
│
├── scan
│   ├── Step 1
│   ├── Step 2
│   └── Step 3
│
├── plan
│   ├── Step 1
│   ├── Step 2
│   └── Step 3
│
└── apply
    ├── Step 1
    ├── Step 2
    └── Step 3
```

---

# 1️⃣ Workflow Name

```yaml
name: Multistage Workflow Tutorial
```

`name` gives the workflow a human-readable name.

GitHub displays this name in the **Actions** section of the repository.

---

# 2️⃣ Workflow Trigger

```yaml
on:
  push:
```

This tells GitHub:

> Run this workflow whenever a push occurs.

For example:

```bash
git add .
git commit -m "Update workflow"
git push
```

The push triggers the workflow.

---

# 3️⃣ Jobs

```yaml
jobs:
```

The `jobs` section contains the individual jobs that make up the workflow.

Our workflow contains:

```yaml
jobs:
  scan:
  plan:
  apply:
```

Each job represents a unit of work that runs on a runner.

---

# 🔍 Job 1 — Scan

```yaml
scan:
  name: Scan Stage
  runs-on: ubuntu-latest
```

There are two important concepts here.

### Job ID

```yaml
scan:
```

`scan` is the **job ID**.

This ID is important because other jobs can reference it:

```yaml
needs: scan
```

### Display Name

```yaml
name: Scan Stage
```

This is the human-readable name displayed in GitHub Actions.

Therefore:

```yaml
scan:
  name: Scan Stage
```

means:

```text
Job ID       → scan
Display Name → Scan Stage
```

---

# 🖥️ `runs-on`

```yaml
runs-on: ubuntu-latest
```

This tells GitHub which runner environment should execute the job.

Our jobs intentionally use different operating systems:

| Job   | Runner          |
| ----- | --------------- |
| Scan  | `ubuntu-latest` |
| Plan  | `macos-latest`  |
| Apply | `ubuntu-latest` |

This demonstrates that different jobs can execute on different runner environments.

---

# 📁 `defaults.run.working-directory`

The Scan job contains:

```yaml
defaults:
  run:
    working-directory: test_environment/dev
```

This establishes the default working directory for the `run` steps in this job.

Therefore:

```yaml
- name: Step 1
  run: echo "Step 1 of Scan Stage"
```

conceptually executes from:

```text
test_environment/dev
```

And:

```yaml
- name: Step 3
  run: |
    cat main.tf
```

reads:

```text
test_environment/dev/main.tf
```

because `main.tf` exists in the configured working directory.

---

# 📥 Why Do We Need `actions/checkout`?

Each job contains:

```yaml
- name: Checkout Repository
  uses: actions/checkout@v7
```

This is extremely important.

When a GitHub-hosted runner starts a job, you should not assume that your repository files are already present in the runner's workspace.

The checkout action checks out the repository so that files such as:

```text
test_environment/dev/main.tf
test_environment/preprod/main.tf
test_environment/prod/main.tf
```

are available.

The sequence is therefore:

```text
Runner starts
     ↓
Checkout repository
     ↓
Repository files available
     ↓
Working directory exists
     ↓
run steps execute
```

---

# ⚠️ What Happens If We Don't Checkout?

Suppose we have:

```yaml
defaults:
  run:
    working-directory: test_environment/dev
```

but don't use:

```yaml
uses: actions/checkout@v7
```

GitHub attempts to start the shell inside:

```text
test_environment/dev
```

If that directory does not exist on the runner, the step can fail with an error similar to:

```text
No such file or directory
```

This is why checkout must normally happen before steps that operate on repository files.

---

# 🔍 Scan Job — Steps

## Step 1

```yaml
- name: Step 1
  run: echo "Step 1 of Scan Stage"
```

This executes in:

```text
test_environment/dev
```

because of the job-level default.

---

## Step 2

```yaml
- name: Step 2
  run: |
    echo "Step 2 of the scan running of linux"
    echo "the scan stage running inside test_environment/dev"
```

The `|` allows multiple shell commands to be written as a multiline block.

Conceptually:

```bash
echo "..."
echo "..."
```

Both commands execute in:

```text
test_environment/dev
```

---

## Step 3

```yaml
- name: Step 3
  run: |
    cat main.tf
    echo "Scan successful"
```

Because the working directory is:

```text
test_environment/dev
```

the command:

```bash
cat main.tf
```

reads:

```text
test_environment/dev/main.tf
```

---

# 📋 Job 2 — Plan

The Plan job contains:

```yaml
plan:
  name: Plan Stage
  needs: scan
  runs-on: macos-latest
```

There are two important concepts here:

```yaml
needs: scan
```

and:

```yaml
runs-on: macos-latest
```

---

# 🔗 Understanding `needs`

```yaml
needs: scan
```

means:

> The Plan job depends on the Scan job.

Therefore GitHub creates this dependency:

```text
scan
  │
  │ needs
  ↓
plan
```

Plan will normally wait until Scan completes successfully.

---

# 🧠 `needs` Creates the Dependency Graph

Without `needs`, independent jobs can potentially run in parallel.

For example:

```text
scan ──────────┐
               │
plan ──────────┤
               ↓
             apply
```

But with:

```yaml
plan:
  needs: scan
```

we explicitly create:

```text
scan
 ↓
plan
```

---

# 📁 Plan Working Directory

The Plan job uses:

```yaml
defaults:
  run:
    working-directory: test_environment/preprod
```

Therefore its normal `run` steps execute from:

```text
test_environment/preprod
```

---

# 📝 Plan Step 1

```yaml
- name: Step 1
  run: |
    echo "Step 1 of the plan running of mac"
    echo "the scan stage running inside test_environment/preprod"
```

These commands execute from:

```text
test_environment/preprod
```

The second message should conceptually be understood as:

```text
The PLAN stage is running inside test_environment/preprod
```

---

# 📄 Plan Step 2

```yaml
- name: Step 2
  run: |
    cat main.tf
    echo "Step2 successful. Directory is plan default directory"
```

Because of:

```yaml
defaults:
  run:
    working-directory: test_environment/preprod
```

this reads:

```text
test_environment/preprod/main.tf
```

---

# 🔄 Step-Level `working-directory`

Now we intentionally override the job default:

```yaml
- name: Step 3
  working-directory: test_environment/prod
  run: |
    cat main.tf
    echo "Step3 successful. Running directory is changed to step working directory"
```

The Plan job's default is:

```text
test_environment/preprod
```

But Step 3 specifies:

```text
test_environment/prod
```

Therefore Step 3 runs from:

```text
test_environment/prod
```

---

# 🥇 Which `working-directory` Wins?

There are two levels:

```yaml
defaults:
  run:
    working-directory: test_environment/preprod
```

and:

```yaml
working-directory: test_environment/prod
```

The more specific step-level configuration wins.

Therefore:

```text
PLAN JOB

Default
   ↓
preprod

Step 1 → preprod
Step 2 → preprod
Step 3 → prod      ← override
```

This is an important GitHub Actions concept.

---

# 🚀 Job 3 — Apply

The Apply job is:

```yaml
apply:
  name: Apply Stage
  if: github.ref == 'refs/heads/main'
  needs: [plan, scan]
  runs-on: ubuntu-latest
```

This demonstrates two important features:

```yaml
needs: [plan, scan]
```

and:

```yaml
if: github.ref == 'refs/heads/main'
```

---

# 🔗 Multiple Job Dependencies

The Apply job uses:

```yaml
needs: [plan, scan]
```

This means Apply depends on both jobs.

Conceptually:

```text
       scan
      /    \
     ↓      ↓
   plan   apply
     \      ↑
      └─────┘
```

More simply:

```text
scan ───────┐
            ├──→ apply
plan ───────┘
```

Apply waits for both dependencies to complete successfully.

---

# 🤔 Why Include `scan` If Plan Already Needs Scan?

Plan has:

```yaml
needs: scan
```

So the dependency chain is already:

```text
scan → plan
```

And Apply has:

```yaml
needs: [plan, scan]
```

This explicitly declares that Apply requires both Scan and Plan.

For this tutorial, this is useful because it demonstrates **multiple job dependencies**.

In a simple linear workflow, you could also write:

```yaml
apply:
  needs: plan
```

which would give:

```text
scan → plan → apply
```

because Plan already depends on Scan.

Both approaches are valid; the current tutorial intentionally demonstrates the array syntax.

---

# 🛑 Conditional Apply

The Apply job contains:

```yaml
if: github.ref == 'refs/heads/main'
```

This checks the Git reference associated with the workflow run.

If the workflow is running from:

```text
refs/heads/main
```

the Apply job can run.

If the workflow is running from another branch:

```text
refs/heads/feature/workflow
```

the Apply job is skipped.

---

# 🌿 Branch-Based Behavior

Suppose we push to:

```text
feature/workflow
```

The workflow becomes:

```text
┌─────────────┐
│ Push branch │
│ feature/... │
└──────┬──────┘
       ↓
     Scan
       ↓
     Plan
       ↓
 Apply → SKIPPED
```

But if we push to:

```text
main
```

we get:

```text
┌─────────────┐
│ Push branch │
│    main     │
└──────┬──────┘
       ↓
     Scan
       ↓
     Plan
       ↓
     Apply
```

This is the foundation of a common CI/CD design:

```text
Feature Branch
      ↓
    Scan
      ↓
    Plan
      ↓
   Review
      ↓
    Merge
      ↓
     Main
      ↓
    Apply
```

---

# 📁 Apply Working Directory

Apply uses:

```yaml
defaults:
  run:
    working-directory: test_module/test_resource1
```

Therefore its normal steps execute from:

```text
test_module/test_resource1
```

---

# 📄 Apply Step 2

```yaml
- name: Step 2
  run: |
    cat main.txt
    echo "Step 2 successful and the working directory is test_resource1"
```

The command:

```bash
cat main.txt
```

therefore reads:

```text
test_module/test_resource1/main.txt
```

---

# 🔄 Apply Step 3 — Override Again

Step 3 changes the directory:

```yaml
- name: Step 3
  working-directory: test_module/test_resource2
  run: |
    cat main.txt
```

So:

```text
Apply default:
test_module/test_resource1

Step 1 → resource1
Step 2 → resource1
Step 3 → resource2
```

This reinforces the same precedence concept demonstrated in the Plan job.

---

# 🧠 The Most Important Concept — Jobs Are Isolated

One of the most important things to understand about GitHub Actions is:

> **Different jobs do not share the same runner filesystem by default.**

Consider:

```text
SCAN JOB
Ubuntu Runner
│
├── checkout
├── dev/
└── scan files
```

Then:

```text
PLAN JOB
macOS Runner
│
├── checkout
├── preprod/
└── plan files
```

Then:

```text
APPLY JOB
Ubuntu Runner
│
├── checkout
├── test_module/
└── apply files
```

These are separate runner environments.

---

# ❌ `needs` Does NOT Share Files

A common misunderstanding is:

```text
scan
 ↓ needs
plan
```

means Plan receives everything that Scan created.

It does **not**.

`needs` establishes a **dependency relationship**.

It does not automatically transfer:

* files
* directories
* environment variables
* installed software
* shell state
* Terraform state
* generated reports

between jobs.

---

# 📦 How Do We Transfer Files Between Jobs?

If a future workflow needs to transfer a file between jobs, GitHub Actions provides mechanisms such as **artifacts**.

For example:

```text
Build Job
   │
   │ upload artifact
   ↓
Artifact
   │
   │ download artifact
   ↓
Deploy Job
```

This becomes particularly important for Terraform workflows where you might want:

```text
Terraform Plan
      ↓
Generate tfplan
      ↓
Upload tfplan as artifact
      ↓
Apply Job
      ↓
Download tfplan
      ↓
terraform apply tfplan
```

That is a natural next step after understanding this tutorial.

---

# 🧩 Complete Dependency Graph

Putting everything together:

```text
                         WORKFLOW
                            │
                            ↓
                      ┌───────────┐
                      │   SCAN    │
                      │  Ubuntu   │
                      │           │
                      │ dev/      │
                      └─────┬─────┘
                            │
                          needs
                            ↓
                      ┌───────────┐
                      │   PLAN    │
                      │   macOS   │
                      │           │
                      │ preprod/  │
                      │           │
                      │ Step 3 →  │
                      │   prod/   │
                      └─────┬─────┘
                            │
                            │
                    ┌───────┴────────┐
                    │                │
                  needs            needs
                    │                │
                    ↓                ↓
                 ┌─────────────────────┐
                 │        APPLY        │
                 │       Ubuntu        │
                 │                     │
                 │ test_module/        │
                 │                     │
                 │ Only on main        │
                 └─────────────────────┘
```

---

# 📊 Working Directory Summary

| Job      | Default Directory            | Override                              |
| -------- | ---------------------------- | ------------------------------------- |
| 🔍 Scan  | `test_environment/dev`       | None                                  |
| 📋 Plan  | `test_environment/preprod`   | Step 3 → `test_environment/prod`      |
| 🚀 Apply | `test_module/test_resource1` | Step 3 → `test_module/test_resource2` |

---

# 📊 Workflow Concepts Summary

| Concept               | Example                          | Purpose                               |
| --------------------- | -------------------------------- | ------------------------------------- |
| Workflow name         | `name:`                          | Name the workflow                     |
| Trigger               | `on: push`                       | Decide when workflow starts           |
| Job ID                | `scan:`                          | Identify a job                        |
| Job name              | `name: Scan Stage`               | Human-readable job name               |
| Runner                | `runs-on:`                       | Select execution environment          |
| Checkout              | `actions/checkout`               | Make repository files available       |
| Job default           | `defaults.run.working-directory` | Set default directory for `run` steps |
| Step override         | `working-directory:`             | Override directory for one step       |
| Dependency            | `needs: scan`                    | Make one job wait for another         |
| Multiple dependencies | `needs: [plan, scan]`            | Wait for multiple jobs                |
| Condition             | `if:`                            | Control whether a job runs            |
| Branch check          | `github.ref`                     | Identify the Git reference            |

---

# 🧪 Things to Experiment With

Once the basic workflow is working, try changing one thing at a time.

### Experiment 1 — Remove checkout

Remove:

```yaml
- uses: actions/checkout@v7
```

Observe what happens when a `run` step tries to use the repository directory.

---

### Experiment 2 — Change the working directory

Change:

```yaml
working-directory: test_environment/dev
```

to:

```yaml
working-directory: test_environment/preprod
```

Observe which `main.tf` is displayed.

---

### Experiment 3 — Override the directory

Try:

```yaml
- name: Test Directory
  working-directory: test_environment/prod
  run: pwd
```

Then compare it with:

```yaml
- name: Default Directory
  run: pwd
```

You should see two different directories.

---

### Experiment 4 — Remove `needs`

Remove:

```yaml
needs: scan
```

from Plan.

Observe the dependency graph.

The Plan job is no longer explicitly dependent on Scan.

---

### Experiment 5 — Change the branch condition

Change:

```yaml
if: github.ref == 'refs/heads/main'
```

to another branch reference and observe when Apply executes.

---

### Experiment 6 — Add another job

Try adding:

```text
test
```

between Scan and Plan:

```text
scan → test → plan → apply
```

This is a useful exercise for understanding dependency graphs.

---

# 🎯 Key Takeaways

If you remember only a few things from this tutorial, remember these:

### 1. Workflow

The workflow is the overall automation definition.

```text
Workflow
```

### 2. Job

A workflow contains jobs.

```text
Workflow
 ├── Scan
 ├── Plan
 └── Apply
```

### 3. Step

A job contains steps.

```text
Job
 ├── Step 1
 ├── Step 2
 └── Step 3
```

### 4. `runs-on`

Determines the runner environment.

```yaml
runs-on: ubuntu-latest
```

### 5. `checkout`

Makes repository files available to the job.

```yaml
uses: actions/checkout@v7
```

### 6. `defaults.run.working-directory`

Sets the default directory for `run` steps.

```yaml
defaults:
  run:
    working-directory: test_environment/dev
```

### 7. Step-level `working-directory`

Overrides the job default for one step.

```yaml
working-directory: test_environment/prod
```

### 8. `needs`

Creates dependencies between jobs.

```yaml
needs: scan
```

or:

```yaml
needs: [plan, scan]
```

### 9. `if`

Controls whether a job or step executes.

```yaml
if: github.ref == 'refs/heads/main'
```

### 10. Jobs are isolated

Each job gets its own runner environment.

```text
Scan Runner ≠ Plan Runner ≠ Apply Runner
```

`needs` controls **execution order**, not filesystem sharing.

---

# 🚀 What's Next?

After understanding this workflow, the next concepts to learn are:

```text
1. ✅ Workflow / Jobs / Steps
       ↓
2. ✅ needs & dependency graphs
       ↓
3. ✅ working-directory
       ↓
4. 🔜 Job Outputs
       ↓
5. 🔜 Artifacts
       ↓
6. 🔜 Environment Variables
       ↓
7. 🔜 GitHub Environments
       ↓
8. 🔜 Secrets
       ↓
9. 🔜 OIDC Authentication
       ↓
10. 🔜 Terraform Plan Artifact
       ↓
11. 🔜 Manual Approval
       ↓
12. 🚀 Production CI/CD Pipeline
```

The concepts in this repository form the foundation for building a real-world **Terraform CI/CD pipeline with GitHub Actions**.
