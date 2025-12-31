# Github Actions
An autonomous system that runs code in your Github repo when something (trigger) happens. That trigger can be: 
1. Code pushes on various branches
2. Pull requests (Open, update, reopen)
3. Manual triggers (pressing the "Run workflow" button)

* The code that is run in these triggers is called workflow.
* The workflow is mainly written as a YAML file inside `.github/workflows/` directory.
* The directory can have multiple YAMLs, each denoting one workflow.
## 1. Top level keys of YAML
### 1.1. name
* Just the name of the workflow.
* This is displayed on the Github UI and has no functional effects in the workflow. 
* The name should be descriptive so that user can know what is going on.

### 1.2. on
* This defines when to run the workflow.
* The most commonly used values for the `on` key are:
```yaml
on: push
on: pull_request
on: workflow_dispatch
```
* `on` can also run on schedules (cronjobs-like)
```yaml
on:
  schedule:
    - cron: "0 0 * * *"
```
* `on` can also be used to do Workflow-to-workflow chaining (triggered when another workflow finishes)

**Example use:**
- CI finishes → CD starts
- Tests pass → Deploy runs

### 1.3. jobs
`jobs` key defines WHAT runs. A workflow has one or more jobs. Each job runs on a fresh machine. Jobs run in parallel by default.
```yaml
jobs: # header
  hello: # identifier of the job, can be anything unique in a scope.
    runs-on: ubuntu-latest
```
### 1.3.1. Runners: `runs-on`

```runs-on: ubuntu-latest```
This tells Github to spin up a VM with Ubuntu installed.
### 1.3.2. Steps:
A job is basically a list of steps, run top to bottom.
**Syntax:**
```yaml
steps:
- name: Build Docker image # Name of the step; displayed on Github UI
  run: docker build -t fastapi-cicd . # Step to execute.
```
Two types of steps:
- `run`: shell commands (as seen above; shell commands depend on the OS we specify on `runs-on`.)
- `uses`: reusable actions

### 1.3.2.1. Uses
Uses means "download some other actions and execute that". When we do:
```yaml
- uses: actions/checkout@v4
```
You're saying: run code that lives in a Github repository called `actions/checkout`, at version `v4`
* A Github Action is simply a Github repo with a special structure designed to be reused in workflows.
* You can find actions and their documentation on `Github Actions Marketplace`.
* You should always look for `Github-maintained actions` before looking for other's actions.
* To know more about some commonly used actions, you can checkout the [Actions Readme](ACTIONS.md).
* Some actions like `appleboy/scp-action@v0.1.4` require arguments (this particular one requires server's host as `host`, username of the admin as `username`, ssh private key as `key`, files from Github's vm to copy as `source` and server's path to paste the copied files as `target`). We provide these arguments with the `with` key. Here's the syntax:
```yaml
- name: Copy image to server
  uses: appleboy/scp-action@v0.1.4
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    source: fastapi-cicd.tar.gz, docker-compose.yml
    target: "~/apps/fastapi-cicd"
```
### 1.4. Secrets
Secrets live in ```Repo → Settings → Secrets → Actions```
Usage:
```yaml
env:
  API_KEY: ${{ secrets.API_KEY }}
```

### 1.5. Job dependencies (`needs`)
Normally, jobs run parallelly. To force order, i.e. to make a job run after one job completes, you can use `needs`
**Usage:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest

  deploy:
    needs: build
    runs-on: ubuntu-latest

```

### 1.6. Caching
When a Github Actions workflow runs:
* Github creates a brand-new virtual machine
* That machine has nothing from previous runs
* When the job finshes, the machine is destroyed.
* Every Python dependencies, Node modules are re-downloaded; every docker layers are rebuilt; build tools reinstall again and again.
* This is **slow** and **wasteful**.
Caching solves this.
#### What shall be cached?
| Language | What to cache             |
| -------- | ------------------------- |
| Python   | pip packages              |
| Node     | node_modules or npm cache |
| Java     | Maven / Gradle cache      |
| Rust     | cargo registry            |
| Docker   | build layers              |

You **do NOT cache**:
* source code
* secrets
* build outputs meant for deployment.

#### How to cache?
Cachine utilizes `actions/cache` Github action.
This action does two things:
1. Tries to restore a cache at the start.
2. Saves a cache at the end (if needed)

**Usage:**
Caching python dependencies:
```yaml
- name: Cache pip dependencies
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-

```
