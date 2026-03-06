---
name: memos-model-switch
description: Switch MemOS chat/reader model safely in a Docker deployment. Use when the user wants to change MemOS model routing, edit the deployment env file, recreate the MemOS container so env_file changes take effect, and verify the effective runtime env and startup health.
---

# MemOS Model Switch

Use this skill when the user wants to change the model used by a MemOS deployment, especially chat/reader routing such as:
- `MOS_CHAT_MODEL=text`
- `MEMRADER_MODEL=text`

This skill is now written as a **general procedure** rather than hardcoding one machine layout.

## Goals

1. Find the active MemOS deployment layout
2. Identify the correct env file and compose file
3. Change only the requested model-routing values
4. Recreate the MemOS service so env changes actually apply
5. Verify the effective runtime env from the running container
6. Check logs for obvious startup or routing problems

## Typical variables

Common variables to inspect or change:
- `MOS_CHAT_MODEL`
- `MEMRADER_MODEL`
- `OPENAI_API_BASE`
- `MEMRADER_API_BASE`
- `ENABLE_CHAT_API`

Usually:
- change `MOS_CHAT_MODEL=<target>`
- keep `MEMRADER_MODEL` aligned with it unless the user explicitly wants a split configuration

Do **not** change API bases, API keys, or unrelated settings unless the user explicitly asks.

## Discovery procedure

Before editing anything, detect the actual deployment paths and names.

### 1) Find likely MemOS repo locations

Check common locations such as:
- `/root/MemOS`
- `$HOME/MemOS`
- current workspace or user-provided path

Look for markers like:
- `.env`
- `docker/docker-compose.yml`
- `docker/compose.yml`
- a compose service named `memos`

### 2) Identify the compose file

Common candidates:
- `<repo>/docker/docker-compose.yml`
- `<repo>/docker/compose.yml`
- `<repo>/docker-compose.yml`
- `<repo>/compose.yml`

### 3) Identify env wiring

Inspect the compose file and determine whether it uses:
- `env_file: ../.env`
- `env_file: .env`
- `environment:` overrides

Important: if the compose stack uses `env_file`, editing the env file alone is not enough. The container must be recreated.

### 4) Identify service/container names

Prefer the compose **service name** for `docker compose up`.

Then detect the runtime **container name** from:
- `container_name:` in compose, or
- `docker ps`

Do not assume names unless verified.

## Update procedure

1. Inspect current config first.
   - Read the env file
   - Confirm the current values for the variables relevant to the request

2. Edit the env file.
   - Set `MOS_CHAT_MODEL=<target>`
   - If appropriate, also set `MEMRADER_MODEL=<target>`
   - Preserve unrelated settings exactly

3. Recreate the MemOS service.
   - Run from the compose directory
   - Preferred command:
     - `docker compose up -d --force-recreate <service>`

4. Verify effective runtime env from the running container.
   - Example:
     - `docker inspect <container> --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -E '^(MOS_CHAT_MODEL|OPENAI_API_BASE|OPENAI_API_KEY|MEMRADER_MODEL|MEMRADER_API_BASE|MEMRADER_API_KEY|ENABLE_CHAT_API)='`

5. Check logs.
   - Example:
     - `docker logs --tail 80 <container>`
   - Look for:
     - startup failures
     - repeated tracebacks
     - bad model-routing errors
     - missing env/config warnings that matter

## Reporting format

Report back with a concise operational summary:
- detected repo path
- detected compose file
- detected service/container name
- old value -> new value
- whether `MEMRADER_MODEL` was also changed
- whether recreate succeeded
- whether runtime env matches expectation
- whether logs look healthy or need follow-up

## Operational rules

- Prefer `docker compose up -d --force-recreate <service>` over plain restart when env changed.
- If multiple MemOS deployments exist, stop and ask which one to modify unless the user already made it clear.
- If compose contains `environment:` overrides for the same variable, account for that before claiming the env file controls the final value.
- Do not leak secrets when reporting output. Summarize sensitive values instead of pasting full keys/tokens.
- If the user asks for “llm-free text auto-routing”, usually set both:
  - `MOS_CHAT_MODEL=text`
  - `MEMRADER_MODEL=text`
  unless they explicitly want chat/reader split.

## Known local example

One known local deployment follows this pattern:
- repo: `/root/MemOS`
- env file: `/root/MemOS/.env`
- compose file: `/root/MemOS/docker/docker-compose.yml`
- service: `memos`
- container: `memos-api-docker`

Use that only when discovery confirms it.
