# Collectu plugins for Claude Code

This repository is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
(`.claude-plugin/marketplace.json` at its root) with two plugins:

| Plugin | Brings |
|---|---|
| `collectu` | The MCP server of the Collectu Hub, signed in with your Collectu account, and the skills below |
| `collectu-core` | The MCP server of a Collectu Core you run - its modules, its running pipeline, its logs and metrics |

Install `collectu` to work with the hub, `collectu-core` as well if you run a Core yourself. Each
works without the other.

## Installing

```bash
claude plugin marketplace add core4x/collectu-claude-plugin
```

```bash
claude plugin install collectu@collectu
```

```bash
claude plugin install collectu-core@collectu
```

In a session, `/plugin marketplace add core4x/collectu-claude-plugin` and `/plugin install collectu@collectu`
do the same. `/plugin marketplace update collectu` fetches new versions.

## Signing in to the hub

The hub's MCP server takes no request without a sign-in, and Claude Code signs in by OAuth: run
`/mcp`, select `plugin:collectu:collectu-hub` and **Authenticate**. Collectu opens in the browser -
sign in, choose whether Claude Code acts for your own account or for your organization and which
permissions it gets, and select **Allow**. Claude Code holds the session from then on; it appears
under **Account > API access tokens** as a connected app, and deleting it there signs Claude Code
out. Your Claude account and your Collectu account are independent of each other.

The plugin connects to the hub of collectu.de, `https://api.collectu.de/mcp/v1`. The address is
fixed so that every Claude client signs in to the same server. For a hub of your own, add its MCP
server yourself and sign in to it the same way, through `/mcp`:

```bash
claude mcp add --transport http collectu-hub-own https://<api address of your hub>/mcp/v1
```

## Connecting a Core

1. Enable the Core's MCP server: `mcp = 1` in its `settings.ini`, then restart it. It is served
   at `http://<host>:<api_port>/mcp/v1/`.
2. `/plugin configure collectu-core@collectu` and set:
   - **Core MCP server** - `http://localhost:8181/mcp/v1/` for a Core on this machine.
   - **Hub API access token** - an [API access token](https://collectu.de/account/api-tokens) of
     the Core's owner. `app:read` lets it in at all and reads the logs, metrics and the running
     pipeline; `module:read` the modules and the validation, `configuration:read` the configuration
     files, and `task:create` starts and stops modules and pipelines. The Core asks the hub what the
     token may do, so it has to be able to reach the hub. Stored in the system's credential store,
     not in `settings.json`. Leave it empty for a Core with `api_authentication = 0`.

Without a token, a Core with authentication lists its tools but answers every call with 401.

A Core speaks plain http: the token travels unencrypted to any host but `localhost`. Reach a Core on
another machine through the vpn of the hub or a tls proxy. For more than one Core, add the others
yourself:

```bash
claude mcp add --transport http collectu-core-line2 http://10.8.0.12:8181/mcp/v1/ --header "Authorization: Bearer <hub api access token>"
```

## Skills

Claude picks a skill when the task matches it; `/collectu:<skill>` starts one by hand.

| Skill | For |
|---|---|
| `module-development` | Writing, reviewing and fixing modules - the contract, the lifecycle, async methods, testing, sending them to the hub |
| `pipeline-configuration` | Building, changing, validating and deploying configurations |
| `core-diagnostics` | Status, logs, running pipeline and packages of Cores; what log lines mean; tasks |
| `data-queries` | Data on Cores (data schemas, data queries), in hosted time-series databases and in data objects |
| `grafana-dashboards` | Panels and dashboards in the hosted Grafana |

The hub also keeps articles in the `Skill` category - procedures written for agents. Its MCP server
tells Claude to look for one before working out a procedure itself.

## Maintaining

- **Raise `version` in a plugin's `plugin.json` with every change.** Claude Code keeps users on the
  version they installed until it changes.
- **This repository is the source of truth for the skills**, `module-development` included. The
  modules repository (core4x/modules) keeps no skill of its own; its contributors get this one
  through the `collectu` plugin. What only applies there - its test kit, layers and CI - is in
  `plugins/collectu/skills/module-development/references/modules-repository-tests.md`, which the
  skill reads only when it works in that repository. Change the module contract here, and the
  test kit's reference whenever the test kit of the modules repository changes.
- Check the marketplace and both plugins before pushing:

  ```bash
  claude plugin validate .
  ```

  ```bash
  claude plugin validate plugins/collectu
  ```

  ```bash
  claude plugin validate plugins/collectu-core
  ```

## License

[MIT](LICENSE) - for both plugins.
