# Jenkins CLI

Node.js/TypeScript Jenkins CLI, command alias `jkc`.

For non-interactive AI Agent usage, see [docs/AI_AGENT.md](docs/AI_AGENT.md).

## Setup

Interactive first-time setup:

```bash
npm install
npm run build
node dist/cli/index.js setup
```

Non-interactive setup:

```bash
npm install
npm run build
node dist/cli/index.js setup \
  --profile local \
  --url http://localhost:8080 \
  --username cli-user \
  --token-env JENKINS_TOKEN \
  --non-interactive
```

Other non-interactive token input:

```bash
printf '%s' '<jenkins-api-token>' | node dist/cli/index.js setup \
  --profile local \
  --url http://localhost:8080 \
  --username cli-user \
  --token-stdin \
  --non-interactive
```

Configuration is stored at:

```text
~/.jenkins-cli/config.json
```

Environment variables can override config:

```text
JENKINS_URL
JENKINS_USERNAME
JENKINS_TOKEN
JENKINS_CRUMB_ISSUER=false
```

Validate the saved profile:

```bash
node dist/cli/index.js config validate
```

Update the current profile interactively:

```bash
node dist/cli/index.js config edit
```

## Commands

```bash
jkc
jck
jkc me
jkc me --mine-limit 50
jkc perms
jkc perms --job builder-pipeline-job
jkc config list
jkc config show
jkc config use local
jkc config validate
jkc config edit
jkc jobs
jkc jobs --search pipeline
jkc jobs --recursive
jkc job builder-pipeline-job
jkc params builder-pipeline-job
jkc nodes
jkc nodes --label linux-builder
jkc node linux-builder
jkc plugins
jkc plugins --search workflow
jkc queue
jkc queue cancel 123
jkc build builder-pipeline-job --param BUILD_TARGET=cli-test --watch
jkc build builder-pipeline-job --label linux-builder --watch
jkc build builder-pipeline-job --node linux-builder --watch
jkc build builder-pipeline-job --node linux-builder --node-param BUILD_NODE
jkc rebuild builder-pipeline-job 3 --watch
jkc build-info builder-pipeline-job 3
jkc status builder-pipeline-job 3
jkc changes builder-pipeline-job 3
jkc steps builder-pipeline-job 3
jkc pipeline-steps builder-pipeline-job 3
jkc log builder-pipeline-job
jkc log builder-pipeline-job 1 --follow
jkc log builder-pipeline-job 1 --output builder-pipeline-job-1.log
jkc log builder-pipeline-job 1 --download
jkc log builder-pipeline-job 1 --download console.log
jkc node-log linux-builder
jkc node-log linux-builder --output linux-builder.log
jkc stop builder-pipeline-job 2
jkc stop builder-pipeline-job --last
jkc stop builder-pipeline-job --running
jkc artifacts builder-pipeline-job
jkc artifacts builder-pipeline-job --download
jkc workspace builder-pipeline-job
jkc workspaces builder-pipeline-job --build 3
jkc workspace builder-pipeline-job build/local-builder-demo.jar --output local-builder-demo.jar
jkc restart-stage builder-pipeline-job 3 --stage "Simulate Build"
```

Running `jkc` or `jck` without subcommands prints a status dashboard with the JKC banner, current login status, config JSON path, current profile, Jenkins URL, username, token mask, and crumb setting.

All query commands support JSON output:

```bash
jkc nodes --json
jkc node linux-builder --json
jkc perms --job builder-pipeline-job --json
```

Most list/report commands support table, JSON, CSV, and Markdown table output:

```bash
jkc jobs --format csv
jkc nodes --format md
jkc queue --format csv
jkc plugins --format csv
jkc artifacts builder-pipeline-job --format md
jkc changes builder-pipeline-job --format md
jkc steps builder-pipeline-job --format csv
jkc workspace builder-pipeline-job --format md
jkc perms --job builder-pipeline-job --format csv
jkc me --format md
```

## Web UI Feature Mapping

```text
Status / View Build Information  jkc build-info <job> [buildNo], jkc status <job> [buildNo]
Changes                          jkc changes <job> [buildNo]
Console Output                   jkc log <job> [buildNo] --follow
Parameters                       jkc params <job>, jkc build-info <job> [buildNo]
Rebuild                          jkc rebuild <job> [buildNo]
Pipeline Steps                   jkc steps <job> [buildNo], jkc pipeline-steps <job> [buildNo]
Workspaces                       jkc workspace <job> [path], jkc workspaces <job> [path]
Previous build                   jkc build-info <job> [buildNo]
Restart from Stage               jkc restart-stage <job> <buildNo> --stage <name>
```

`restart-stage` depends on Jenkins Declarative Pipeline restart support and only works for builds where Jenkins exposes that action. `steps` uses the Pipeline REST API when present and falls back to parsing console output when the API is unavailable.

Supported formats:

```text
table
json
csv
md
```

## Exit Codes

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

## Local Jenkins Test Profile

The local Docker Jenkins environment created for development has:

```text
baseUrl:  http://localhost:8080
username: cli-user
node:     linux-builder
job:      builder-pipeline-job
```

## `jkc me` Job History

`jkc me` shows:

- current Jenkins user and instance security status
- running builds started by the current user
- jobs that the current user has triggered in recent build history

Vanilla Jenkins does not expose the exact job creator through the standard REST API. Exact creator lookup requires audit/history plugins. Use `--mine-limit <count>` to control how many recent builds per job are scanned.
