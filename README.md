# Three decisions before you self-host an open-source AI employee

This repository is about running an open-source AI employee on infrastructure you control, and about the three decisions that come before any command: where the agent runs, where the credentials live, and what stops a write. Kortix is the open-source AI Operating System; agents, skills, connectors and company memory live as files you can inspect. The job an agent holds can be support triage, a weekly report, or sales ops, and the platform underneath that job can run on your own server.

The deployment is a single Compose stack, set up and kept current by the `kortix self-host` commands, and it uses the release images Kortix runs in its own cloud. The database, stored files, repos and policies live on disks you control, and one instance directory holds the three things you back up: `.env`, the Postgres data directory and the storage directory.

## The three decisions

| Decision | What it settles | Where the answer changes |
| --- | --- | --- |
| Where the agent runs | The sandbox provider, and whether the instance is reachable from the internet | A laptop can run the stack, but sessions need a public API |
| Where the credentials live | Connector credentials, project secrets and model keys | Connector grants are per project; model keys are per project too |
| What stops a write | The `policy` block in `kortix.yaml` | You set allow, ask or block per action |

Each one has a different answer for a laptop, a VPC and an on-prem network. The documents in `docs/` work through them in order, then the first run, then the credential and approval configuration an operator does before inviting anyone.

## What runs where

Every service belongs to one Compose project named `kortix-<instance>`. The server runs the browser app, the REST API, the model gateway, a one-shot migration job, the updater, the Supabase set, and Caddy when a domain is configured. Two things always sit off the box:

- **Sandboxes.** A session runs on Platinum (the default; Daytona is a legacy fallback), created with your provider key, and reaches your instance over a public URL. The API has to be addressable from outside for any of this to start.
- **Model providers.** The gateway on your box calls the providers your projects connect.

Three Docker Hub repositories supply it: `kortix/kortix-frontend`, `kortix/kortix-api` and `kortix/kortix-gateway`. A pull needs no credentials.

## Quick start

On a Linux host that has a public IP, DNS already pointed at it, and Docker with the Compose plugin installed:

```bash
curl -fsSL https://kortix.com/install | bash
kortix self-host init --domain kortix.example.com --admin-email you@example.com
kortix self-host start
```

On a bare server, Kortix publishes a bootstrap script in [the repository](https://github.com/kortix-ai/suna) that installs Docker and the CLI, runs `kortix self-host init`, sets the domain and starts the stack; [the self-hosting page](https://kortix.com/docs/host) lists its exact invocation and flags. If the script added you to the `docker` group, open a new shell, then:

```bash
kortix self-host configure
kortix self-host start
kortix self-host doctor
```

## Before you invite anyone

Claim the admin account with an address listed in `KORTIX_PLATFORM_ADMIN_EMAILS`. Connect GitHub at `/admin/git`, build the team account and a project, and add the model key under Customize, Models, Providers. Set `EMAIL_URL` or `DISABLE_SIGNUP=true` early, since otherwise anyone who can reach the instance may create a confirmed account for any address, an admin address included.

## In this repository

- [`docs/01-what-self-hosting-means.md`](docs/01-what-self-hosting-means.md): the three decisions, compared across a laptop, a VPC and on-prem.
- [`docs/02-first-run.md`](docs/02-first-run.md): the documented first run, command by command, with a checkpoint after each step.
- [`docs/03-credentials-and-approvals.md`](docs/03-credentials-and-approvals.md): where each class of credential lives and how an action is allowed, asked or blocked.
- [`docs/04-faq.md`](docs/04-faq.md): the questions an IT reviewer asks about data location, models, audit and upgrades.

## The short version of the security story

One isolated sandbox per session. Each session has its own isolated machine and branch. Connector credentials are brokered server-side and never enter the machine. Every action is recorded. Reading, exporting and streaming the audit log depend on the plan. The rule that stops a write is a policy block in `kortix.yaml`, committed with the rest of the project, so a diff shows who changed what an agent may do.

For the wider question of running an AI employee as a role, see [opensourceaiemployee.com](https://opensourceaiemployee.com). The [open-source AI employee guide](https://opensourceaiemployee.com/open-source-ai-employee) is the long version of that case. The product documentation is at [kortix.com/docs](https://kortix.com/docs).
