# Databricks bundles: Terraform engine vs direct engine

The code and commands behind my article: **[Terraform engine vs direct engine: what Databricks bundles really remember](link)**

Databricks is moving bundles from the Terraform deployment engine to the new direct engine. This repository is a small lab to see what actually changes: one serverless job, deployed with both engines side by side.

## What's inside

- `databricks.yml`: the bundle, with two identical targets, `tf` (Terraform engine) and `direct` (direct engine)
- `resources/my_job.yml`: the job
- `src/`: a tiny Python package, built as a wheel on every deploy
- The Git history: one commit per step, including the deployment state in `.databricks/`. Use `git log --oneline` and `git show <commit>` to follow the experiments.

## Before you start

- All commands were run on Windows, in PowerShell, with Databricks CLI 1.2.1 (later upgraded to 1.26.0) on Databricks Free, in November 2026.
- Replace `<your-workspace>` and `<your-email>` with your own values.
- The experiments only deploy jobs and never run them. Section 9 removes everything from your workspace.

# Commands to repeat the experiments

---

## 0. Prerequisites

You need:

- a Databricks workspace (Databricks Free is enough)
- [uv](https://docs.astral.sh/uv/), which the bundle uses to build the wheel
- Git
- Databricks CLI **1.2.1**, the last version where the Terraform engine is the default

```powershell
# install a specific CLI version with winget
winget install Databricks.DatabricksCLI --version 1.2.1
databricks --version                       # Databricks CLI v1.2.1

# log in to your workspace
databricks auth login --host https://<your-workspace>.cloud.databricks.com
```

If winget doesn't offer 1.2.1, download it from the [CLI releases page](https://github.com/databricks/cli/releases/tag/v1.2.1).

---

## 1. Get the code

Clone the repository and start your own branch from the first commit, before any deploy. The later commits contain my deployment state, which points to jobs in my workspace.

```powershell
git clone <repository-url>
cd databricks-bundle-engines
git switch -c my-run (git rev-list --max-parents=0 HEAD)
```

In `databricks.yml`, set `host` in both targets to your workspace URL, and commit the change.

---

## 2. Validate

```powershell
databricks bundle validate -t tf
databricks bundle validate -t direct

# which Terraform and provider versions this CLI will download
databricks bundle debug terraform
```

---

## 3. First deploy with both engines

The engine is chosen per command. The `tf` target uses the default Terraform engine. The `direct` target uses the direct engine through an environment variable.

```powershell
# Terraform engine (the first deploy also downloads Terraform and the provider)
Measure-Command { databricks bundle deploy -t tf | Out-Default }

# direct engine
$env:DATABRICKS_BUNDLE_ENGINE = "direct"
Measure-Command { databricks bundle deploy -t direct | Out-Default }
Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

# check that the direct engine did the deploy: there must be no terraform folder here
Get-ChildItem -Recurse -Name .databricks\bundle\direct
```

Run both deploys a second time, without any changes, to compare the timing without the downloads.

> If a Terraform-engine deploy fails with `could not connect to registry.terraform.io`, run it again. `terraform init` contacts the registry on every deploy, even when the provider is already downloaded.

### Commit the state

The CLI puts a `.gitignore` file in `.databricks/` and `dist/`, so Git ignores both folders. For this experiment you want to see every change, so remove them, and use `git add -f` from now on, because the CLI can create the `.gitignore` file again. Don't do this in a real project.

```powershell
Remove-Item .databricks\.gitignore, dist\.gitignore
git add -A; git add -f .databricks
git commit -m "state after the first deploy"
```

---

## 4. Compare the first deploy

### Locally

```powershell
Get-ChildItem -Recurse -Name .databricks\bundle\tf
Get-ChildItem -Recurse -Name .databricks\bundle\direct

# the two state files
(Get-Content .databricks\bundle\tf\terraform\terraform.tfstate).Count
(Get-Content .databricks\bundle\direct\resources.json).Count
```

Then open `terraform.tfstate` and `resources.json` side by side in your editor.

### In the workspace

```powershell
# job IDs and workspace paths
databricks bundle summary -t tf
$env:DATABRICKS_BUNDLE_ENGINE="direct"; databricks bundle summary -t direct; Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

# save the job IDs for the next steps
$tfJob     = "<job id from the tf summary>"
$directJob = "<job id from the direct summary>"

# the remote state folders
databricks workspace list "/Workspace/Users/<your-email>/.bundle/my-bundle/tf/state"
databricks workspace list "/Workspace/Users/<your-email>/.bundle/my-bundle/direct/state"

# compare the two deployed jobs
databricks jobs get $tfJob     | Out-File -Encoding utf8 tf-job.json
databricks jobs get $directJob | Out-File -Encoding utf8 direct-job.json
git diff --no-index tf-job.json direct-job.json
Remove-Item tf-job.json, direct-job.json
```

> PowerShell's `>` saves files as UTF-16, and `git diff` then treats them as binary. Use `| Out-File -Encoding utf8` instead.

---

## 5. Experiment 1: add and remove `timeout_seconds`

`timeout_seconds` is a plain optional field of a job. Its default is `0`, which means no timeout.

### Add it

In `resources/my_job.yml`, add one line under `name: my_job`:

```yaml
      timeout_seconds: 3600
```

```powershell
git commit -am "config: add timeout_seconds"

databricks bundle deploy -t tf
$env:DATABRICKS_BUNDLE_ENGINE="direct"; databricks bundle deploy -t direct; Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

databricks jobs get $tfJob     | Select-String timeout_seconds     # 3600
databricks jobs get $directJob | Select-String timeout_seconds     # 3600

git add -A; git add -f .databricks
git commit -m "deploy: timeout_seconds added"

# how each engine recorded the new value
git diff HEAD~1 HEAD -- .databricks/bundle/tf/terraform/terraform.tfstate
git diff HEAD~1 HEAD -- .databricks/bundle/direct/resources.json
```

### Remove it

Delete the `timeout_seconds: 3600` line again.

```powershell
git commit -am "config: remove timeout_seconds"

# what each engine plans to do
New-Item -ItemType Directory -Force .plans | Out-Null
databricks bundle plan -t tf
git diff -- .databricks/bundle/tf/terraform/bundle.tf.json      # the field is gone from the generated config

$env:DATABRICKS_BUNDLE_ENGINE="direct"
databricks bundle plan -t direct
databricks bundle plan -t direct -o json | Out-File -Encoding utf8 .plans\direct-plan.json
Remove-Item Env:DATABRICKS_BUNDLE_ENGINE
Select-String -Path .plans\direct-plan.json -Pattern timeout -Context 3

# deploy and check
databricks bundle deploy -t tf
$env:DATABRICKS_BUNDLE_ENGINE="direct"; databricks bundle deploy -t direct; Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

databricks jobs get $tfJob     | Select-String timeout_seconds     # 0
databricks jobs get $directJob | Select-String timeout_seconds     # 0

git add -A; git add -f .databricks
git commit -m "deploy: timeout_seconds removed"

git diff HEAD~1 HEAD -- .databricks/bundle/tf/terraform/terraform.tfstate
git diff HEAD~1 HEAD -- .databricks/bundle/direct/resources.json
```

To read the binary Terraform plan file, you can try `terraform show` with the Terraform binary the CLI downloaded (not tested in the article):

```powershell
.databricks\bundle\tf\bin\terraform.exe "-chdir=.databricks\bundle\tf\terraform" show plan
```

---

## 6. Find the fields Terraform treats as computed

Terraform can print the provider schema without going to the internet:

```powershell
.databricks\bundle\tf\bin\terraform.exe "-chdir=.databricks\bundle\tf\terraform" providers schema -json | Out-File -Encoding utf8 .plans\provider-schema.json
```

In the file, look at `provider_schemas` → `resource_schemas` → `databricks_job`. Attributes with both `"optional": true` and `"computed": true` are the ones Terraform keeps when you remove them from the config. For a job, the top-level ones are only `format` and `id`. Inside a task, `retry_on_timeout` is the one you can use with a serverless wheel task.

---

## 7. Experiment 2: add and remove `retry_on_timeout`

### Add it

In `resources/my_job.yml`, add one line to the task:

```yaml
      tasks:
        - task_key: hello
          environment_key: default
          retry_on_timeout: true
```

```powershell
git commit -am "config: add retry_on_timeout"

databricks bundle deploy -t tf
$env:DATABRICKS_BUNDLE_ENGINE="direct"; databricks bundle deploy -t direct; Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

databricks jobs get $tfJob     | Select-String retry_on_timeout    # true
databricks jobs get $directJob | Select-String retry_on_timeout    # true

git add -A; git add -f .databricks
git commit -m "deploy: retry_on_timeout added"
```

### Remove it

Delete the `retry_on_timeout: true` line again.

```powershell
git commit -am "config: remove retry_on_timeout"

databricks bundle plan -t tf                                   # 0 to change
git diff -- .databricks/bundle/tf/terraform/bundle.tf.json      # the field is gone, but Terraform plans nothing

$env:DATABRICKS_BUNDLE_ENGINE="direct"
databricks bundle plan -t direct                                # 1 to change
databricks bundle plan -t direct -o json | Out-File -Encoding utf8 .plans\direct-plan-retry.json
Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

databricks bundle deploy -t tf
$env:DATABRICKS_BUNDLE_ENGINE="direct"; databricks bundle deploy -t direct; Remove-Item Env:DATABRICKS_BUNDLE_ENGINE

databricks jobs get $tfJob     | Select-String retry_on_timeout    # still true
databricks jobs get $directJob | Select-String retry_on_timeout    # no line, so false (the default)

git add -A; git add -f .databricks
git commit -m "deploy: retry_on_timeout removed"
```

The `tf` job now has `retry_on_timeout: true` in the workspace and nothing about it in the bundle. That's the starting point for the migration.

---

## 8. Upgrade the CLI and migrate the `tf` target

**Don't deploy the `tf` target with the new CLI before running `migrate`.** From CLI 1.14, a deploy migrates a Terraform-engine target on its own.

```powershell
# upgrade (the article used 1.26.0)
winget upgrade Databricks.DatabricksCLI
databricks --version

databricks bundle validate -t tf
```

### Step 1: migrate the state (local only)

```powershell
databricks bundle deployment migrate -t tf
```

This creates `.databricks/bundle/tf/resources.json` and renames `terraform.tfstate` to `terraform.tfstate.backup`. Nothing changes in the workspace yet. To undo it, delete `resources.json` and rename the backup back to `terraform.tfstate`. The command prints the exact paths.

```powershell
git add -A; git add -f .databricks
git commit -m "migrate: tf target to direct engine (local state only)"
git show --stat HEAD
```

### Step 2: compare the migrated state with the direct one

```powershell
git diff --no-index .databricks/bundle/tf/resources.json .databricks/bundle/direct/resources.json
```

### Step 3: the plan before the first direct deploy

```powershell
databricks bundle plan -t tf
databricks bundle plan -t tf -o json | Out-File -Encoding utf8 .plans\tf-plan-after-migrate.json
Select-String -Path .plans\tf-plan-after-migrate.json -Pattern retry_on_timeout -Context 3
```

In the article, this plan wasn't empty. It planned to reset `retry_on_timeout`.

### Step 4: the first deploy with the direct engine

```powershell
databricks bundle deploy -t tf
databricks jobs get $tfJob | Select-String retry_on_timeout        # no line, so it's back to false
databricks workspace list "/Workspace/Users/<your-email>/.bundle/my-bundle/tf/state"

git add -A; git add -f .databricks
git commit -m "deploy: tf target on direct engine after migration"
```

---

## 9. Clean up

```powershell
databricks bundle destroy -t tf --auto-approve
databricks bundle destroy -t direct --auto-approve
```
