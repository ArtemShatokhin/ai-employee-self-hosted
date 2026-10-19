# Questions an IT reviewer asks about an open-source AI employee

Short answers about an open-source AI employee you host yourself, with the documented behaviour behind each one. Where the self-hosting page does not cover a subject, this document says so rather than guessing.

## Where the data sits

The instance directory on your server holds the Postgres data directory, the storage directory and the `.env` that carries every key the instance uses. Back up those three and you have backed up the instance. Session work runs on a sandbox at the provider you chose, so the compute and the files a session touches live with that provider for the life of the machine; only what the agent commits survives it. Model calls leave your network for whichever provider the project connected, because the gateway on your box calls that provider. No self-hosted region control is documented beyond the experimental `us_region` flag, which sets Platinum sandbox placement only and does not move the API, the database or the archive.

## Model providers you connect

Any provider whose key you hold. The self-hosted page names Anthropic, OpenAI, Google, Groq, xAI, DeepSeek, Mistral, Bedrock and OpenRouter, plus the ChatGPT plan you already hold, connected with `kortix providers login chatgpt`. Each project carries its own key, and every model call routes through the gateway inside your stack, so Kortix has no credential in that path.

## Audit access

Every action is recorded. Reading, exporting and streaming the audit log depend on the plan. On a self-hosted instance, audit read is available with an Enterprise license, which also unlocks SSO, SCIM and role-based access control. The connector gateway writes the record for every call that reaches a connected tool, including the action, the agent, the outcome, and a hash of the arguments rather than the arguments themselves.

## Upgrades and rollback

`kortix-updater` checks once a day at `KORTIX_UPDATE_TIME`, which defaults to 02:00 in `KORTIX_UPDATE_TZ`, default `America/New_York`. It checks free disk space, stops below `KORTIX_MIN_FREE_DISK_MB` (2048 by default), pulls the configured tags, runs `kortix-migrate`, then rolls the API, gateway and frontend. A migration that fails swaps nothing, and a rollout that fails keeps the previous version serving. Three tracks exist: `stable`, `latest`, and an exact `--tag`, and a pin wins over the channel.

```bash
kortix self-host update
kortix self-host update --tag <version>
kortix self-host rollback --release <version>
kortix self-host version
```

## Backup and restore

Stop the stack so the database files are consistent, copy the instance directory, then start again:

```bash
kortix self-host stop
sudo tar -czf kortix-backup.tgz -C ~/.config/kortix/self-host default
kortix self-host start
```

To restore, put the directory back in the same place and run `kortix self-host start`. There is no separate backup service to configure, and nothing to export from Kortix.

## Air-gapped networks

No, and it should not be described that way. The start command pulls images from Docker Hub, and each session runs at a remote sandbox provider that calls back to your instance's public API URL. A network that blocks both cannot run sessions. Isolated topologies are scoped with Kortix rather than self-served.

## Account creation on your instance

Only two settings. Until `EMAIL_URL` is set or `DISABLE_SIGNUP=true`, anyone who reaches the instance can create a confirmed account for any address, including an address on `KORTIX_PLATFORM_ADMIN_EMAILS`, and whoever signs up first with that address becomes the admin. Set `EMAIL_URL` first, because it also turns off automatic email confirmation and restores invite matching, then close sign-up once the team is in.

## Ports and exposure

In domain mode only Caddy publishes public ports, 80 and 443. Every other published port binds `127.0.0.1`. In local mode, with no domain and no tunnel, the frontend answers on 13737, the API on 13738 and Supabase on 13740, and sessions do not run because the sandbox cannot reach the API. Supabase Studio is reachable only through an SSH tunnel to `127.0.0.1:13740`.

## Several instances on one server

Yes, with `--instance <name>` on every command. Each instance gets its own directory and its own Compose project. Only one instance per server can run in domain mode, because Caddy binds ports 80 and 443.

## Docker and a domain

The bootstrap runs on Linux and installs Docker Engine and the Compose plugin when they are missing. The CLI itself runs on Linux and macOS, on x86_64 and arm64. A domain is not required for evaluation: tunnel mode serves the web app at `http://localhost:13737` and publishes only the API through Cloudflare, though a quick tunnel address changes each time `cloudflared` restarts and breaks registered OAuth callbacks.

## Sandbox sharing

One isolated sandbox per session. Each session has its own isolated machine and branch. Connector credentials are brokered server-side and never enter the machine. The database enforces the one-machine-per-session rule, so the same mechanism separates two of your own sessions and two different customers.
