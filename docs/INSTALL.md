# jenkins-cli Install Guide

Concise install path for human and AI agent use.

## 1. Install

```bash
npm install -g https://github.com/fightmonster/jenkins-cli/releases/latest/download/jenkins-cli.tgz
```

The `latest/download` URL always points to the most recent release; no version pinning needed.

## 2. Generate an API Token

Go to:

```text
<jenkins-url>/user/<your-username>/security/apiToken
```

Click "Add new Token", give it a name, copy the value. You will not see it again.

If that page 404s (Jenkins < 2.129), go to:

```text
<jenkins-url>/user/<your-username>/security/
```

and find the API Token section.

## 3. Configure

```bash
JENKINS_TOKEN=<your-api-token> jkc setup \
  --url <jenkins-url> \
  --username <your-username> \
  --token-env JENKINS_TOKEN \
  --non-interactive
```

This writes a profile to `~/.jenkins-cli/config.json` (mode `0600`). The `JENKINS_TOKEN` env var is consumed by `setup` only and discarded; subsequent `jkc` calls read from the config file.

## 4. Verify

```bash
jkc me
```

Expected output includes your username, the Jenkins URL, and `Authenticated: yes`.

## Token from stdin (alternative)

If you prefer not to set an env var even for one command:

```bash
printf '%s' '<your-api-token>' | jkc setup \
  --url <jenkins-url> \
  --username <your-username> \
  --token-stdin \
  --non-interactive
```

## Switching Profiles

```bash
jkc -j <profile-name> me     # one-call override
JENKINS_PROFILE=<name> jkc me
jkc config use <name>        # makes it the default
```

## Uninstall

```bash
npm uninstall -g jenkins-cli
rm -rf ~/.jenkins-cli         # remove stored profiles and token
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No Jenkins profile configured` | No `setup` run yet, no env vars | Run step 3 |
| `Jenkins API failed: HTTP 401 Unauthorized` | Token wrong, revoked, or user has no access | Regenerate token, verify user has API access in Jenkins |
| `Jenkins API failed: HTTP 403 Forbidden` | Token valid but user lacks the specific capability (e.g. `Overall/Administer` for `plugins`) | Use a different account or skip the command |
| `Profile not found: <name>` | `-j` or `JKC_PROFILE` points to a non-existent profile | Run `jkc config list` to see available profiles, or re-run `setup` with `--profile <name>` |
| setup prompts interactively | Missing `--non-interactive` or required flag | Add `--non-interactive`, `--url`, `--username`, and a token source |
