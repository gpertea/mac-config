# Multiple Slack workspaces in the Codex desktop app

Configure one named MCP server per Slack workspace, then authorize each name separately. Both use `https://mcp.slack.com/mcp`; the workspace is selected during Slack OAuth consent. Server names label connections; they do not select a workspace by themselves.

The app can call tools from both connections within one local chat. No separate Slack browser tabs, exported messages, local Slack proxy, or background service are required for this method. Each connection acts with the account and permissions authorized for it.

## Prerequisites

- Codex desktop app and CLI on the same local host, using the same Codex configuration directory (`~/.codex` by default).
- The recipient's own Slack membership and permission to authorize the integration in each workspace.
- An approved Slack MCP integration and its supplied configuration. Its OAuth client ID may already be supplied by the integration or an existing working configuration; ordinary users need not create a Slack app or obtain a new client ID.

All client IDs below are placeholders. They are valid TOML strings but cannot authenticate. No working client ID, user identity, workspace ID, or credentials are included.

### What login supplies automatically

The OAuth client ID identifies the integration application. Login obtains and stores the user's access credentials after workspace consent; those credentials must never be copied from another user. In the verified installation, the client ID was already present in imported server configuration before `codex mcp login` ran. No manual client-ID entry was needed during login.

A supplied integration can handle its application configuration automatically. This is different from assuming that a bare `codex mcp login` creates a Slack application or adds a fixed Slack client ID to `config.toml`. Codex supports automatic client registration for compatible servers, but Slack documents that Dynamic Client Registration is unsupported. Keep a supplied client ID when present; do not replace it with a placeholder.

The verified setup uses a pre-registered client ID without a configured client secret. A different Slack app may require additional OAuth configuration, a secret, and registered callback URLs; follow that app's maintainer instructions.

### Optional: obtaining a client when none is supplied

Use this section only if an approved integration has not already supplied a working configuration. Request its public client ID and supported redirect configuration from the maintainer. A client ID identifies an application, not a user's authorization; it does not grant workspace access. Workspace membership alone does not supply an OAuth client registration.

If no approved integration is available, an authorized app maintainer can prepare one:

1. Create an internal Slack app for the target workspace, or use an eligible published integration. For unrelated workspaces, separate internal apps and client IDs may be needed.
2. Obtain workspace approval for MCP access. Copy the app's **Client ID** from Slack app settings; do not confuse it with an app ID, workspace ID, client secret, or access token.
3. Configure the required **User Token Scopes**. Public-channel listing needs `channels:read`; public message search needs `search:read.public`; public channel history needs `channels:history`. Add other scopes only for intended capabilities. A maintainer's existing integration may already define its scopes.
4. Configure desktop OAuth and the exact callback URI used by Codex. Register the callback reported by `codex mcp add`; do not guess its path. If a provider requires a fixed listener port, configure `oauth.callback_port` consistently with its registered redirect. Retry login after registration.
5. For a new public desktop client, review Slack's PKCE setup. Enabling PKCE is a one-way app setting according to Slack; choose it deliberately on an app intended for desktop use. An existing confidential client instead needs its maintainer's supported secret provisioning procedure.

This app-registration route is provider-dependent and has not been tested from scratch. Do not assume any arbitrary Slack client ID will work. See [Slack MCP application eligibility](https://docs.slack.dev/ai/slack-mcp-server/), [Slack desktop OAuth with PKCE](https://docs.slack.dev/authentication/using-pkce/), and [Codex callback configuration](https://learn.chatgpt.com/docs/extend/mcp?surface=cli).

## 1. Inspect and preserve existing configuration

Run in a local terminal:

```sh
command -v codex
codex --version
codex mcp add --help
codex mcp list
```

The CLI must support `--url` and `--oauth-client-id`. If `codex` is unavailable, use the desktop app's CLI installation option if provided, or follow the official CLI installation documentation. Do not replace a working installation unnecessarily.

Preserve the existing configuration before editing:

```sh
if [ -f "$HOME/.codex/config.toml" ]; then
    (umask 077; cp "$HOME/.codex/config.toml" \
      "$HOME/.codex/config.toml.before-multi-slack-$(date +%Y%m%d-%H%M%S)")
fi
```

Keep this backup local: an existing configuration may contain unrelated credentials. Use unused server names; preserve existing Slack connections.

## 2. Configure one alias per workspace

### Existing or supplied configuration: no manual ID entry

Start with a working Slack MCP entry supplied by the approved integration. In the local `config.toml`, duplicate its non-secret connection settings under two unused names, such as `slack_a` and `slack_b`. Copy its Slack URL and public OAuth client configuration, including `client_id` if present. Preserve any provider-required callback settings. Never copy authorization headers, bearer tokens, client secrets, authentication files, or another user's login state.

Keep the original entry if it is still needed. The two aliases can use the same supplied client ID when the approved application supports both workspaces. Different approved applications need their respective supplied configurations.

Proceed directly to section 3. This is the configuration-import method used in the verified installation.

### Optional CLI setup when configuring the ID explicitly

Replace the client ID placeholders below with the approved application ID. Use different IDs if the workspaces require different approved applications.

Run the first command and finish any OAuth flow it opens:

```sh
codex mcp add slack_a --url https://mcp.slack.com/mcp \
  --oauth-client-id 'REPLACE_WITH_WORKSPACE_A_APPROVED_CLIENT_ID'
```

In Slack's authorization page, sign in with the recipient's own account and select workspace A. If adding the server does not complete authentication, use the login commands in section 3.

After the first authorization succeeds, configure workspace B:

```sh
codex mcp add slack_b --url https://mcp.slack.com/mcp \
  --oauth-client-id 'REPLACE_WITH_WORKSPACE_B_APPROVED_CLIENT_ID'
```

Select workspace B explicitly; the browser may initially show workspace A. If needed, complete authentication in section 3.

The relevant configuration has this shape; merge it rather than replacing the whole file:

```toml
[mcp_servers.slack_a]
url = "https://mcp.slack.com/mcp"

[mcp_servers.slack_a.oauth]
client_id = "REPLACE_WITH_WORKSPACE_A_APPROVED_CLIENT_ID"

[mcp_servers.slack_b]
url = "https://mcp.slack.com/mcp"

[mcp_servers.slack_b.oauth]
client_id = "REPLACE_WITH_WORKSPACE_B_APPROVED_CLIENT_ID"
```

These mocked `client_id` values illustrate where an integration-supplied ID belongs. They are not IDs returned by login and must not be pasted over working supplied values. If a supported integration manages client registration without these explicit tables, retain its configuration instead of adding mocked tables.

OAuth credentials are generated locally through login. Do not copy another user's credentials, Codex authentication files, keychain entries, or browser sessions.

## 3. Authorize the aliases separately

For configured aliases that are not yet authenticated, run one command at a time:

```sh
codex mcp login slack_a
```

Complete workspace A's Slack consent with the user's own account. Then run:

```sh
codex mcp login slack_b
```

Choose workspace B explicitly and complete consent. If `mcp add` already authenticated an alias successfully, its login need not be repeated. Login stores the resulting credentials locally; no manual token handling is required.

## 4. Reload and verify in the desktop app

Run `codex mcp list` to confirm both entries exist. Reload MCP servers in the app's settings, or restart the app when ongoing work permits, then open a new local Codex chat.

Use this verification prompt:

> Using slack_a and slack_b separately, list up to two public channels from each connection. Confirm the workspace mapping with a public channel unique to each workspace. Perform read-only checks.

Success requires two functioning connections with the intended workspace identities. Seeing `general` and `random` twice is insufficient to establish that the workspaces differ. Tool namespaces should distinguish the server names, such as `mcp__slack_a__...` and `mcp__slack_b__...`.

Then requests can target either workspace or both in the same chat. Include workspace names when channel names overlap.

For example:

> Search public channels in both workspaces for release-planning messages from the past week. Group the results by workspace and link each source message.

For three or more workspaces, add a new alias and complete a separate OAuth flow for each one. Verify every mapping before relying on combined results.

## Troubleshooting and removal

- **Both connections reach one workspace:** run `codex mcp logout slack_b`, then `codex mcp login slack_b` and select the correct workspace.
- **Registration, client, or redirect error:** verify the approved client ID and callback requirements with the integration maintainer. Retry with the correct configuration; do not extract browser tokens as a workaround.
- **Configured but absent in the app:** verify the app and terminal use the same local host and Codex configuration directory, then reload and use a new chat. Local settings do not automatically configure cloud chats or another computer.
- **Permission or scope error:** check workspace approval and the authorized application's scopes.

To remove only the added connections:

```sh
codex mcp logout slack_a
codex mcp remove slack_a
codex mcp logout slack_b
codex mcp remove slack_b
```

Reload the app afterward. Remove only aliases created for this setup.

## Verification scope and sharing

Two configured Slack MCP connections returned public-channel listings in one local chat on 2026-10-12. Configuration was inspected on 2026-10-13; both used the same endpoint and client ID. Setup records confirmed that non-secret connection settings, including the client ID, were imported before separate OAuth logins. CLI syntax was checked against `codex-cli 0.162.0-alpha.17.2`. Fresh installation, custom app registration, and another user's authorization have not been tested.

This file can be handed off independently. Repository access supplies documentation, not Slack authorization. The recipient must authorize both workspaces independently. Keep local configuration backups and generated credentials outside the repository.

## References

- [OpenAI: Codex MCP configuration and OAuth](https://learn.chatgpt.com/docs/extend/mcp?surface=cli)
- [Slack: connecting MCP to agent harnesses](https://docs.slack.dev/ai/slack-mcp-server/connect-to-harnesses/)
- [Slack: MCP endpoint, application eligibility, and authentication](https://docs.slack.dev/ai/slack-mcp-server/)
- [Slack: desktop OAuth with PKCE](https://docs.slack.dev/authentication/using-pkce/)

Command syntax follows the inspected CLI and OpenAI documentation; Slack's harness guide currently shows a different `mcp add` syntax.
