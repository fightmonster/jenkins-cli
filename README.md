# jenkins-cli

Node.js / TypeScript command-line client for Jenkins, with explicit support for AI agents and automation.

- Package name: `jenkins-cli`
- Binaries: `jenkins-cli`, `jkc`, `jck` (all point to the same entry)
- Node: `>= 20.0.0`
- License: ISC

For non-interactive AI agent usage, see [docs/AI_AGENT.md](docs/AI_AGENT.md).

## Quick Start

```bash
# 1. Install (always pulls the latest release)
npm install -g https://github.com/fightmonster/jenkins-cli/releases/latest/download/jenkins-cli.tgz

# 2. Configure (one command, token saved to ~/.jenkins-cli/config.json)
JENKINS_TOKEN=<your-api-token> jkc setup \
  --url <jenkins-url> \
  --username <your-jenkins-username> \
  --token-env JENKINS_TOKEN \
  --non-interactive

# 3. Verify
jkc me
```

Generate an API token at: `<jenkins-url>/user/<your-username>/security/apiToken`

## Install

```bash
npm install -g jenkins-cli
# or, from a clone:
npm install
npm run build
npm link          # exposes jenkins-cli / jkc / jck on PATH
```

## First-Time Setup

Interactive:

```bash
jkc setup
```

You will be prompted for:

1. Profile name (e.g. `work`, `local`)
2. Jenkins URL (e.g. `https://jenkins.example.com`)
3. Username
4. API token (or password)

Need a token? Generate one at:

```text
<jenkins-url>/user/<username>/security/apiToken
```

Non-interactive:

```bash
JENKINS_TOKEN=<token> jkc setup \
  --profile work \
  --url <jenkins-url> \
  --username <username> \
  --token-env JENKINS_TOKEN \
  --non-interactive
```

Or read the token from stdin:

```bash
printf '%s' '<token>' | jkc setup \
  --profile work \
  --url <jenkins-url> \
  --username <username> \
  --token-stdin \
  --non-interactive
```

Configuration is stored at `~/.jenkins-cli/config.json` (mode `0600`).

## Authentication Priority

`jenkins-cli` resolves the active profile in this order:

1. `-j <name>` / `--jenkins <name>` (one-shot override)
2. `JKC_PROFILE=<name>` env var
3. `JENKINS_URL` + `JENKINS_USERNAME` + `JENKINS_TOKEN` env trio
4. `currentProfile` in `~/.jenkins-cli/config.json`

The name `env` is reserved and forces use of the env trio (no profile entry required).

```bash
# Use a different profile for one call
jkc -j staging me

# Use the env-trio profile explicitly
JENKINS_URL=<jenkins-url> JENKINS_USERNAME=<user> JENKINS_TOKEN=<token> \
  jkc -j env jobs
```

## Commands

Run `jkc --help` for the live list. The most-used groups:

```text
identity / permissions
  me, perms

configuration
  setup, config (list | show | use | validate | edit)

jobs
  jobs, job, params, job-type

instance
  center (identity | labels), crumb

nodes / builders
  nodes, node, node-log

queue
  queue (read), queue cancel <id>

build (state-modifying)
  build, rebuild, stop, restart-stage

build (read-only)
  build-info | status, changes, steps, log

artifacts / workspace
  artifacts, workspace
```

All query commands support `--json`. Most list / report commands support `-f table|json|csv|md`.

### Examples

```bash
jkc me                                         # who am I, what's my Jenkins
jkc perms                                      # capability inventory
jkc perms --job <job-name>                     # per-job permissions

jkc jobs                                       # list jobs at root
jkc jobs -r                                    # recursive (folder / multibranch)
jkc jobs -s release                            # substring filter
jkc jobs -p "*-black"                          # glob filter
jkc jobs -p "**/test-*" -r                     # recursive glob
jkc job <job-name>                             # job details
jkc params <job-name>                          # parameter definitions

jkc nodes                                      # build machines
jkc node <node-name>                           # single machine
jkc center identity                            # Jenkins mode / version
jkc center labels                              # agent labels
jkc crumb                                      # CSRF crumb (Jenkins-Crumb=...)

jkc build <job-name> -p KEY1=VAL1 -p KEY2=VAL2 # trigger a build
jkc build <job-name> -p K=V --node <name> -w   # trigger + wait + stream log
jkc rebuild <job-name> 42                      # rebuild with #42's parameters
jkc stop <job-name> 42                         # abort build #42
jkc build-info <job-name> 42                   # build summary
jkc log <job-name> 42 -f                       # stream log
jkc log <job-name> 42 -o build-42.log          # save log to file
jkc artifacts <job-name> 42 --download         # download all artifacts
jkc workspace <job-name> path/to/file -o out   # download one workspace file

jkc queue                                      # see what's pending
jkc plugins                                    # installed plugins (needs admin)
```

## Output Formats

```bash
jkc jobs --format csv
jkc nodes --format md
jkc build-info <job> 1 --json
```

Supported by most list / report commands:

```text
table   # default, human-readable
json
csv
md      # markdown table
```

## Exit Codes

```text
0   success
1   command or validation error
2   build result: UNSTABLE
3   build result: FAILURE
4   build result: ABORTED
5   build result: unknown / non-success
10  Jenkins / API / network error
20  config / profile error
```

Use these in scripts to distinguish "build failed" from "could not reach Jenkins".

## Web UI → CLI Mapping

```text
Status / View Build Information   jkc build-info <job> [buildNo]
                                  jkc status   <job> [buildNo]
Changes                           jkc changes  <job> [buildNo]
Console Output                    jkc log <job> [buildNo] --follow
Parameters                        jkc params <job>
                                  jkc build-info <job> [buildNo]
Rebuild                           jkc rebuild <job> [buildNo]
Pipeline Steps                    jkc steps <job> [buildNo]
                                  jkc pipeline-steps <job> [buildNo]
Workspaces                        jkc workspace <job> [path]
                                  jkc workspaces <job> [path]
Previous build                    jkc build-info <job> [buildNo]
Restart from Stage                jkc restart-stage <job> <buildNo>
                                       --stage <name>
```

`restart-stage` requires Declarative Pipeline restart support. `steps` uses the Pipeline REST API when present and falls back to console parsing.

## `jkc me` Job History

`jkc me` reports:

- current Jenkins user and instance security status
- running builds started by the current user
- jobs the current user has triggered in recent build history

Vanilla Jenkins does not expose the exact job creator through the standard REST API. Use `--mine-limit <count>` to control how many recent builds per job are scanned.

## Development

```bash
npm run build     # tsc → dist/
npm run dev       # tsc --watch
npm test          # tsc + node --test dist/**/*.test.js
npm pack          # builds jenkins-cli-<version>.tgz
```

## License

ISC
