# Jenkins CLI AI Agent Guide

This file is the stable, non-interactive usage guide for AI agents calling `jkc`.

## Runtime

```bash
npm install
npm run build
node dist/cli/index.js --help
```

The npm binary name is `jkc` after package install/link. Inside this repo, use:

```bash
node dist/cli/index.js <command>
```

`jck` is also registered as a compatibility alias. Running `jkc`/`jck` without a subcommand prints a human-readable dashboard; agents should prefer explicit commands with `--json`.

## Authentication

Human first-time setup can be interactive:

```bash
node dist/cli/index.js setup
```

Agents should avoid interactive mode. Use environment variables or stdin so secrets are not written in shell history.

```bash
JENKINS_URL=http://localhost:8080 \
JENKINS_USERNAME=cli-user \
JENKINS_TOKEN=<api-token> \
node dist/cli/index.js me --json
```

Persistent profile:

```bash
JENKINS_TOKEN=<api-token> node dist/cli/index.js setup \
  --profile local \
  --url http://localhost:8080 \
  --username cli-user \
  --token-env JENKINS_TOKEN \
  --non-interactive
```

Token from stdin:

```bash
printf '%s' '<api-token>' | node dist/cli/index.js setup \
  --profile local \
  --url http://localhost:8080 \
  --username cli-user \
  --token-stdin \
  --non-interactive
```

Do not call `setup` without `--non-interactive` from an AI Agent process; it may wait for TTY input.

Validate config:

```bash
node dist/cli/index.js config validate --json
node dist/cli/index.js config list --json
node dist/cli/index.js config show --json
```

Humans can update an existing profile interactively:

```bash
node dist/cli/index.js config edit
```

Agents should not use `config edit`; use `setup --non-interactive` instead.

Config file:

```text
~/.jenkins-cli/config.json
```

## Output Contract

Use `--json` for structured output when available. List/report commands also support:

```text
--format table
--format json
--format csv
--format md
```

Exit codes:

```text
0   success
1   command or validation error
2   build result: UNSTABLE
3   build result: FAILURE
4   build result: ABORTED
5   build result: unknown/non-success
10  Jenkins/API/network error
20  config/profile error
```

## Common Calls

Identity and permissions:

```bash
node dist/cli/index.js me --json
node dist/cli/index.js perms --json
node dist/cli/index.js perms --job builder-pipeline-job --json
```

Jobs and parameters:

```bash
node dist/cli/index.js jobs --recursive --json
node dist/cli/index.js job builder-pipeline-job --json
node dist/cli/index.js params builder-pipeline-job --json
```

Nodes/builders:

```bash
node dist/cli/index.js nodes --json
node dist/cli/index.js nodes --label linux-builder --json
node dist/cli/index.js node linux-builder --json
```

Queue:

```bash
node dist/cli/index.js queue --json
node dist/cli/index.js queue cancel <queue-id>
```

Build:

```bash
node dist/cli/index.js build builder-pipeline-job \
  --param BUILD_TARGET=agent-run \
  --label linux-builder \
  --json
```

Watch build logs:

```bash
node dist/cli/index.js build builder-pipeline-job \
  --param BUILD_TARGET=agent-run \
  --label linux-builder \
  --watch
```

Read build status:

```bash
node dist/cli/index.js build-info builder-pipeline-job lastBuild --json
node dist/cli/index.js status builder-pipeline-job 1 --json
```

Stop builds:

```bash
node dist/cli/index.js stop builder-pipeline-job <build-no>
node dist/cli/index.js stop builder-pipeline-job --last
node dist/cli/index.js stop builder-pipeline-job --running
```

Logs:

```bash
node dist/cli/index.js log builder-pipeline-job 1
node dist/cli/index.js log builder-pipeline-job 1 --download build.log
node dist/cli/index.js node-log linux-builder --output linux-builder.log
```

Artifacts and workspace:

```bash
node dist/cli/index.js artifacts builder-pipeline-job 1 --json
node dist/cli/index.js artifacts builder-pipeline-job 1 --download downloads
node dist/cli/index.js workspace builder-pipeline-job --build 1 --json
node dist/cli/index.js workspace builder-pipeline-job build/file.jar --build 1 --output file.jar
```

Pipeline extras:

```bash
node dist/cli/index.js changes builder-pipeline-job 1 --json
node dist/cli/index.js steps builder-pipeline-job 1 --json
node dist/cli/index.js restart-stage builder-pipeline-job 1 --stage "Compile"
```

Plugins:

```bash
node dist/cli/index.js plugins --json
node dist/cli/index.js plugins --search workflow --json
```

Plugin listing usually requires Jenkins administrator permissions.

## Scheduling Notes

Jenkins does not let a CLI override an arbitrary Jenkinsfile `agent` directly. `jkc build --label <label>` and `jkc build --node <node>` inject a parameter named `NODE_LABEL` by default. The job must use it, for example:

```groovy
agent { label "${params.NODE_LABEL}" }
```

Use `--label-param` or `--node-param` when a job expects a different parameter name.
