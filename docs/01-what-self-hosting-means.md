# What self-hosting an open-source AI employee means

Kortix is the open-source AI Operating System. Self-hosting it means running the whole control plane inside your own network: accounts, projects, repos, secrets, connectors, policies and audit, on storage you back up yourself. One command starts the stack, and three separate decisions decide what it is allowed to do.

## Decision one: where the agent runs

The stack and the work are in different places. `kortix self-host start` runs a Docker Compose project, `kortix-<instance>`, on your server: the browser app, the `/v1` API, the model gateway, Supabase, a migration job, an updater, and Caddy when you run domain mode. Every session, the thing that actually executes a turn, runs on a sandbox provider off the box: Daytona by default, or Platinum or E2B. The instance creates it with your provider key.

That split creates one hard requirement. The sandbox has to call back to the instance over the public API URL, held in `KORTIX_URL`. A machine that cannot be reached from the internet cannot host running sessions. The self-host page states this plainly: without a domain, every port binds `127.0.0.1`, and sessions do not run because the sandbox cannot reach the API. This is also why Kortix does not describe the deployment as air-gapped. Images come from Docker Hub, and the default sandbox provider is remote.

For a laptop, that means tunnel mode: `kortix self-host init --tunnel cloudflare`, then `kortix self-host start`, after which `http://localhost:13737` serves the browser app and only the API is published through a quick tunnel. It is an evaluation setup, and a quick tunnel URL changes whenever `cloudflared` restarts, which breaks every OAuth callback registered against it. A named tunnel with `CLOUDFLARE_TUNNEL_TOKEN` and `CLOUDFLARE_TUNNEL_HOSTNAME` keeps the URL stable, and domain mode is the production shape.

## Decision two: where the credentials live

Three credential classes behave differently. Connector credentials are brokered server-side and never enter the machine; the sandbox carries a single project-scoped Kortix token, and the third-party key is attached to one outbound request at a time. Project secrets are delivered into the session at boot, by name, and the default exposure is an environment variable. Model keys are set per project, in the dashboard or with `kortix providers set <provider>`, and every model call routes through the gateway on your box.

The operator question is which value a command the agent runs can read, which value the agent has to name, and which one is decrypted outside the machine entirely. Document 03 works through the order to configure them.

## Decision three: what stops a write

`kortix.yaml` carries a `policy` block that names which tool calls run, which wait for a person, and which are refused. The three actions are `always_run`, `require_approval` and `block`. Rules match a glob over fully qualified tool paths, so one line can gate a single call or a whole connector, and conditions match the arguments. `default_mode: risk` lets reads run and sends writes and destructive calls to a person. A project with no policy block keeps the permissive default, which is why the block is worth committing before the first agent is given a connector. Session work reaches `main` through a change request. Merge is default-deny for agents.

## The same stack in three places

| | Laptop (tunnel) | VPC | On-prem network |
| --- | --- | --- | --- |
| Reachability | Quick or named Cloudflare tunnel | Domain with TLS | Domain with TLS |
| Inbound ports | None; the API is published through the tunnel URL | 80 and 443 | 80 and 443 |
| Sessions run | Yes, on a remote provider | Yes, on a remote provider | Yes, on a remote provider |
| DNS needed | None for a quick tunnel | `<domain>`, `api.<domain>`, `*.p.<domain>` | Same, on internal or split DNS |
| Who runs the box | You | You | You |

The Compose project is identical in all three. What changes is where you point the domain and how much you give it. Isolated topologies sit outside this self-serve path, and Kortix scopes them with the customer rather than publishing a recipe for one. For a team whose constraint is that no workload may leave the network, that is the conversation to have before a trial, since the sandbox provider is the part that sits off the box in every self-served mode.
