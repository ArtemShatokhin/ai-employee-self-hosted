# First run of an open-source AI employee stack

This walkthrough covers a public instance of the open-source Kortix stack in domain mode. Each step says what it produces and what to check before the next one. Every command below appears on Kortix's self-hosting page; where no command is published for a step, the step says so.

## 1. Prepare DNS and ports

Create `A` or `AAAA` records for `<domain>` and `api.<domain>`, both pointed at the server. If you want sandbox previews, add a wildcard `*.p.<domain>` record as well. Open ports 80 and 443 inbound.

Caddy needs those records before the first start, because it obtains the TLS certificate over ACME HTTP-01 on the first request. If you add Apps hosting later, that needs its own wildcard record such as `*.apps.<domain>`.

Check: `dig +short kortix.example.com` and `dig +short api.kortix.example.com` both return the server address, and `curl -I http://kortix.example.com` from outside reaches the host.

## 2. Run the bootstrap

On a bare server, Kortix publishes a bootstrap script in [the repository](https://github.com/kortix-ai/suna) that installs Docker and the Compose plugin, installs the CLI, runs `kortix self-host init`, sets the domain and runs `kortix self-host start`. [The self-hosting page](https://kortix.com/docs/host) gives its exact invocation and the flags it accepts, including `--domain`, `--email`, `--channel`, `--auto-update`, `--instance` and `--daytona-key`; re-running it is safe.

It writes the instance into `~/.config/kortix/self-host/<instance>/`: `docker-compose.yml`, `.env`, a `Caddyfile` and `updater.sh`. Everything you later back up lives in that directory.

Check: `kortix self-host status` reports the stack up with the services listed, and the instance directory exists.

## 3. Open a new shell, then configure

If the bootstrap added your user to the `docker` group, open a new shell first or the next command cannot reach the Docker socket.

```bash
kortix self-host configure
```

`configure` asks the setup questions in a fixed order, each pre-filled with the current value so Enter keeps it:

1. Reachability, either domain (domain, API subdomain, ACME email) or tunnel.
2. Admin email, written to `KORTIX_PLATFORM_ADMIN_EMAILS`.
3. Enterprise license, which turns on SSO, SCIM, role-based access control and audit read on this instance.
4. Restrict account creation, on by default.
5. Sandbox provider: `platinum`, or `daytona` as the legacy fallback, plus that provider's key and settings.
6. Apps hosting, optional.
7. Preview origins, suggested `p.<domain>`, which needs the wildcard record from step 1.
8. Composio, optional, and the app catalog for connectors.
9. Update policy: auto-update on or off, channel, daily time and timezone.

The sandbox provider key is the one credential the stack cannot start without. A model key is not asked here, on purpose.

`configure` writes the files only. Apply them with:

```bash
kortix self-host start
```

Check: `kortix self-host plan` validates the rendered Compose file and applies nothing, and `kortix self-host ps` lists the running services.

## 4. Sign up as the admin

Open `https://kortix.example.com` and sign up with an address in `KORTIX_PLATFORM_ADMIN_EMAILS`. A password, email-code or social sign-in matches that list. A SAML sign-in never does.

Do this as soon as the stack answers. Until `EMAIL_URL` is set or `DISABLE_SIGNUP=true`, anyone who reaches the instance can create a confirmed account for any address, the admin address included. Two commands close that window:

```bash
kortix self-host env set EMAIL_URL=smtp://user:password@smtp.example.com:587
kortix self-host env set DISABLE_SIGNUP=true
kortix self-host start
```

Check: `kortix self-host doctor` reports `open-sign-up` passing.

## 5. Connect GitHub and create the team account

Open `/admin/git` and connect the instance's managed GitHub identity. Until it is set, Kortix cannot create project repositories; an existing repository can still be connected.

Your sign-up account is personal. It cannot hold SAML SSO, and a project cannot move to another account. Create one project at `/new`, then use the project menu, Switch Project, Create an account to make the team account, and create the project there.

Check: `/admin` shows the instance numbers, and the new project opens with an empty repo.

## 6. Add a model key

In the project, open Customize, Models, Providers, or run:

```bash
kortix providers set anthropic sk-ant-...
kortix providers ls
```

A self-hosted instance has no Kortix-managed models, and each project needs its own key.

## 7. Close the run with doctor

```bash
kortix self-host doctor
```

The command checks Docker, the Compose plugin, the rendered Compose file, the instance directory, `EMAIL_URL`, preview origins and open sign-up. It exits 0 when every check passes and 1 when one fails; `--json` prints the same result for a script. Fix each failing check before you invite anyone, because the preview-origins failure is the one that breaks browser previews later.

## Manual path

On a server that already has Docker and the Compose plugin, the documented alternative is three commands:

```bash
curl -fsSL https://kortix.com/install | bash
kortix self-host init --domain kortix.example.com --admin-email you@example.com
kortix self-host start
```

`start` runs `init` first when no configuration exists. If the sandbox key is missing, `init` and `start` print a warning and continue, which is why the provider key is worth setting in `configure` before you rely on sessions starting.
