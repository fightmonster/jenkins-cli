# Jenkins CLI AI Agent Guide

This file is the stable, non-interactive usage guide for AI agents calling `jenkins-cli` (binary alias: `jkc` / `jck`).

The npm package name is `jenkins-cli`. The CLI binary is registered under three names:

```text
jenkins-cli   # canonical / install name
jkc           # short alias (default in --help Usage lines)
jck           # legacy / compatibility alias
```

In commands below, `jkc` is used interchangeably with `jenkins-cli` / `jck`.

## Runtime

After `npm install -g jenkins-cli` (or `npm link` from this repo):

```bash
jenkins-cli --help
jkc --help
jck --help
```

From a clone of this repo, before install:

```bash
npm install
npm run build
node dist/cli/index.js --help
```

Running `jkc` (or `jenkins-cli`) with no subcommand prints a human-readable dashboard; agents should prefer explicit subcommands with `--json` for parseable output.

## Quick Start

```bash
# 1. Install (always pulls the latest release)
npm install -g https://github.com/fightmonster/jenkins-cli/releases/latest/download/jenkins-cli.tgz

# 2. Configure (non-interactive, token saved to ~/.jenkins-cli/config.json)
JENKINS_TOKEN=<api-token> jkc setup \
  --url <jenkins-url> \
  --username <username> \
  --token-env JENKINS_TOKEN \
  --non-interactive

# 3. Verify
jkc me --json
```

The `JENKINS_TOKEN` env var is consumed by `setup` and discarded; future `jkc` calls read from `config.json` and need no env vars.

## Authentication

Three independent auth paths, in priority order:

1. `-j <name>` global flag → switches the active profile for one call only.
2. `JKC_PROFILE=<name>` env var → same as above, env-driven.
3. `JENKINS_URL` + `JENKINS_USERNAME` + `JENKINS_TOKEN` env trio → ephemeral "env" profile.
4. `~/.jenkins-cli/config.json` `currentProfile` → persistent default profile.

> The special name `env` (e.g. `-j env`) explicitly requests the env trio and is required when the env vars are set but no `env` profile exists in `config.json`.

### Recommended: env vars only, no config file

```bash
JENKINS_URL=<jenkins-url> \
JENKINS_USERNAME=<username> \
JENKINS_TOKEN=<api-token> \
jkc me --json
```

Token never written to disk; gone when the shell exits. Preferred for ephemeral / CI / agent contexts.

### Persistent profile (non-interactive)

```bash
JENKINS_TOKEN=<api-token> jkc setup \
  --profile <profile-name> \
  --url <jenkins-url> \
  --username <username> \
  --token-env JENKINS_TOKEN \
  --non-interactive
```

### Persistent profile (stdin)

```bash
printf '%s' '<api-token>' | jkc setup \
  --profile <profile-name> \
  --url <jenkins-url> \
  --username <username> \
  --token-stdin \
  --non-interactive
```

> Agents must always pass `--non-interactive`; without it, `setup` waits for TTY input.

### Profile operations

```bash
jkc config list --json
jkc config show --json
jkc config show <profile-name> --json
jkc config validate --json
jkc config use <profile-name>     # changes currentProfile in config.json
jkc -j <profile-name> me          # one-call profile override
```

### Where the API token comes from

Jenkins → User → Configure → API Token → Add new Token. URL pattern:

```text
<jenkins-url>/user/<username>/security/apiToken
```

Older Jenkins (< 2.129) may only expose:

```text
<jenkins-url>/user/<username>/security/
```

Config file (created by `setup`):

```text
~/.jenkins-cli/config.json
```

Token is stored in plaintext; file mode is `0600`. Treat the file like a credential.

## Output Contract

Always use `--json` for parseable output. Most list/report commands also support:

```text
-f, --format table    # default, human-readable
-f, --format json
-f, --format csv
-f, --format md
```

`--json` and `-f json` are equivalent; use `--json` for agent scripts.

### Exit codes

```text
0   success
1   command or validation error
2   build result: UNSTABLE
3   build result: FAILURE
4   build result: ABORTED
5   build result: unknown/non-success
10  Jenkins / API / network error
20  config / profile error
```

Agents should branch on these codes when chaining commands (e.g. distinguish "build failed" from "could not reach Jenkins").

## Command Reference (all read-only unless noted)

### Identity / permissions

```bash
jkc me [--mine-limit N] [--json]
jkc perms [--job <job-name>] [--json]
```

### Configuration (state-modifying unless noted)

```bash
jkc config list                          # read-only
jkc config show [<profile>]              # read-only
jkc config validate [<profile>]          # read-only
jkc config use <profile>                 # writes config.json
jkc config edit [<profile>]              # interactive only, do not use from agent
jkc setup [...flags] --non-interactive   # writes config.json
```

### Jobs (read-only)

```bash
jkc jobs [-s <keyword>] [-p <glob>] [-r] [--json] [-f <format>]
jkc job <job-name> [--json]
jkc params <job-name> [--json]
jkc job-type [--json]                    # requires Job/Create, agents without perm get a 403 hint
```

`-p` accepts glob: `*` does not cross `/`, `**` crosses `/`, `?` matches one char.

### Instance / crumbs (read-only)

```bash
jkc center identity [--json]
jkc center labels  [--json]
jkc crumb [--json]                       # Jenkins-Crumb=... for custom script integration
```

### Nodes / builders (read-only)

```bash
jkc nodes [--label <label>] [--json]
jkc node <node-name> [--json]
jkc node-log <node-name> [--output <file>]    # requires Agent/Connect or Agent/Configure
```

### Queue

```bash
jkc queue [--json]                              # read-only
jkc queue cancel <queue-id>                     # state-modifying, requires confirmation
```

### Build (state-modifying)

```bash
jkc build <job-name> [-p KEY=VAL ...] [--node <name>] [--label <label>] [--node-param <name>] [--label-param <name>] [-w|--watch] [--json]
jkc rebuild <job-name> [build-no] [-w|--watch] [--json]
jkc stop <job-name> [build-no] [--last|--running]    # state-modifying
jkc restart-stage <job-name> <build-no> --stage <name> [--json]  # state-modifying
```

`-p` may be repeated for multiple build parameters. `--watch` blocks until the build is queued, then streams the console log until completion.

`--node <name>` / `--label <label>` inject the value as a build parameter named `NODE_LABEL` by default; rename with `--node-param` / `--label-param`.

### Build read-only

```bash
jkc build-info | status <job-name> [build-no] [--json]
jkc changes     <job-name> [build-no] [--json]
jkc steps       <job-name> [build-no] [--json]    # Pipeline steps, falls back to console parsing
jkc log         <job-name> [build-no] [-f|--follow] [-o|--output <file>] [--download [name]]
```

`build-no` defaults to `lastBuild`; special values: `lastBuild`, `lastSuccessfulBuild`, `lastFailedBuild`, `lastCompletedBuild`, `lastStableBuild`, `lastUnstableBuild`, `lastUnsuccessfulBuild`, `lastUncompletedBuild`.

### Artifacts / workspace (read-only; download writes to local disk)

```bash
jkc artifacts  <job-name> [build-no] [-d|--download [dir]] [--json]
jkc workspace  <job-name> [workspace-path] [-b|--build N] [-o|--output <file>] [--url] [--json]
```

Download targets:
- `--download [dir]` for artifacts, dir defaults to `./downloads`
- `--output <file>` for a single workspace file
- `--url` prints the workspace URL only

## Global flag

```bash
-j, --jenkins <name>    # one-shot profile override
```

Must be passed before the subcommand: `jkc -j staging jobs`. Mutates `process.env.JKC_PROFILE` for the duration of the call. Does not write to `config.json`.

## Dangerous Commands

The following commands change Jenkins state and **must not be run without explicit user confirmation**:

```text
build           # triggers a new build
rebuild         # triggers a new build
stop            # aborts a build, leaves ABORTED record
restart-stage   # restarts from a stage
build -w        # long-running, also triggers
queue cancel    # cancels queued item
config use      # writes config.json
config edit     # interactive, do not use from agent
setup           # writes config.json
```

Read commands (`me`, `perms`, `jobs`, `job`, `params`, `nodes`, `node`, `queue` without cancel, `build-info`, `changes`, `steps`, `log`, `artifacts --list`, `workspace --list/--url`, `crumb`, `center`, `job-type`, `plugins`, `config list/show/validate`) are safe to run without confirmation.

## Scheduling Notes

`jkc build --node <name>` and `jkc build --label <label>` inject a build parameter named `NODE_LABEL` (rename with `--node-param` / `--label-param`). The job's Jenkinsfile must consume that parameter, for example:

```groovy
agent { label "${params.NODE_LABEL}" }
```

If the job does not use `NODE_LABEL`, the injection is a no-op and the job runs on its default agent.

## Common Patterns

```bash
# Discover the Jenkins instance
jkc me --json
jkc center identity --json
jkc center labels --json

# Discover available jobs
jkc jobs -p "*-release" -r --json
jkc job <job-name> --json
jkc params <job-name> --json

# Discover what an agent can run
jkc nodes --json
jkc nodes --label <label> --json
jkc node <node-name> --json

# Read the latest build state
jkc build-info <job-name> lastBuild --json
jkc log <job-name> lastBuild 200       # last 200 lines

# Trigger a build (after explicit user confirmation)
jkc build <job-name> -p KEY1=VAL1 -p KEY2=VAL2 -w
```
