# Credentials and approvals for an open-source AI employee

An engineer setting up an open-source AI employee on their own infrastructure has four places to look and one file to edit. The four places hold connector credentials, project secrets, model provider keys and the instance's own encryption key. The file is `kortix.yaml`, where the rule that stops a write lives. Getting the order right matters, because a policy committed after an agent has already been handed a connector has protected nothing.

## Where connector credentials live

Connector credentials are brokered server-side and never enter the machine. A connector belongs to the project, so it is connected once and every session that project starts can reach it.

The path of a call is short and one-directional. The agent names the connector and the action, with no URL, host or key. The gateway checks that this agent may use that connector, resolves the policy, decrypts the credential on the server and attaches it to the outbound request. The third-party API sees a normal authenticated request and the response returns to the agent. The credential stays behind.

Each agent lists the connectors it may use, as text in the repo:

```toml
[[agents]]
name = "support"
connectors = ["zendesk", "gmail"]

[[agents]]
name = "reporting"
connectors = ["warehouse"]
```

The reporting agent cannot call Zendesk, and cannot discover that Zendesk exists. Effective access is always the intersection of what the person can do and what the agent was granted, so a grant never lifts a person's own limit.

One setting belongs on the operator's checklist on day one. Connectors refuse private and internal addresses with `connector_egress_blocked`. If an agent must reach an internal service, list the hosts in `KORTIX_CONNECTOR_EGRESS_ALLOW_HOSTS` rather than widening anything globally.

## Where secrets live

Secrets are encrypted at rest with a key per project. Delivery has two gates: the person's role and the agent's declared grant must both allow the value. At boot the secret is placed in the session by name, on tmpfs at mode 0600, and the tool reads it from the environment; it is not written into the prompt. On shutdown the file is wiped and the machine is destroyed with it.

One fact about that mechanism is worth stating to a reviewer. `API_KEY_SECRET` in the instance `.env` encrypts every stored secret, connector credential and token. Lose that value and they become unreadable: the Secrets page marks them Undecryptable, a connector call returns `credential_undecryptable`, and a session start asks for the first value it needs. The CLI refuses to rotate `API_KEY_SECRET`, `POSTGRES_PASSWORD` and `SAML_PRIVATE_KEY` for that reason.

Rotate anything else with:

```bash
kortix self-host env ls
kortix self-host env rotate GITHUB_CLIENT_SECRET
kortix self-host env set KORTIX_CONNECTOR_EGRESS_ALLOW_HOSTS=wiki.internal,db.internal
```

`env ls` masks secrets until you add `--show`. `env set` restarts the services that read the value.

## Where model keys live

A self-hosted instance has no Kortix-managed model lineup. Each project connects its own providers, and the keys are stored as encrypted project secrets:

```bash
kortix providers set anthropic sk-ant-...
kortix providers set openai sk-...
kortix providers login chatgpt
kortix providers ls
```

Every model call routes through the gateway inside your own stack. The gateway resolves the credential and calls the provider, and Kortix has no credential in that path. Supported named providers on the self-hosted page are Anthropic, OpenAI, Google, Groq, xAI, DeepSeek, Mistral, Bedrock and OpenRouter, plus the ChatGPT subscription you already hold.

## What stops a write

Approval gates you set. Off until you set them. The mechanism is a `policy` block in `kortix.yaml`, versioned with the rest of the project:

```yaml
policy:
  default_mode: risk
  policies:
    - match: gmail.send_email
      action: require_approval
      conditions:
        - arg: to
          match: /@example\.com$/
    - match: gmail.send_email
      action: block
    - match: stripe.*
      action: require_approval
```

Three actions exist: `always_run`, `require_approval` and `block`. A rule matches a glob over fully qualified tool paths, so one line covers a single call or a whole connector. When a call is held, a person sees the action and its arguments, and can approve it once, approve it for the rest of the session, or deny it; the run resumes where it stopped rather than failing and retrying. An argument the rule cannot evaluate fails closed. There is no blanket allow-always, because a later call with different arguments would hide behind it.

`default_mode: risk` lets reads run and sends writes and destructive calls to a person. A project with no policy block keeps the permissive default, so an untouched instance runs everything.

## Where the floor is

Session work reaches `main` through a change request. Merge is default-deny for agents. A grant for `project.gitops.merge` changes that for one agent, and it is a line in `kortix.yaml`. Nothing a session does is visible to another session or to `main` until then.

## The order an operator should work in

Seven steps follow the dependencies in the stack, and audit read on a self-hosted instance is available with an Enterprise license.

1. Set `EMAIL_URL`, then `DISABLE_SIGNUP=true`, before the first invitation.
2. Run `kortix self-host doctor` until every check passes.
3. Connect GitHub at `/admin/git`.
4. Add the model key in the project.
5. Add internal hosts to `KORTIX_CONNECTOR_EGRESS_ALLOW_HOSTS` if any connector must reach them.
6. Commit the `policy` block, with `default_mode: risk`, before connecting anything that writes.
7. Connect connectors and narrow each agent's list in `kortix.yaml`.
