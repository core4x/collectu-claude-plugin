---
name: core-diagnostics
description: "Find out what a Collectu Core is doing and why something does not work: its status, logs, running pipeline, module states, throughput, installed packages, vulnerabilities and licenses - through the Collectu Hub or the MCP server of a local Core - and restart, stop, start or update it with a task when the user asks. Use when the user asks how their Cores are doing, why data is missing, or about errors, warnings or a module that does not start."
---

# Diagnosing a Collectu Core

A Core reports to the hub with every heartbeat (every few seconds): its status, its log lines,
the pipeline it runs and its installed packages. The hub answers with what the Core last reported;
a local Core's own MCP server answers with what is true right now.

## Where to look

| Question | Hub | Local Core |
|---|---|---|
| Which Cores are there, which are online, which have errors | `app_get_my` (`status`, `has_error_logs`, `log_count`, throughput) | - |
| One Core | `app_get_by_app_id` | - |
| Its log | `app_get_logs` (as fresh as the last heartbeat) | `get_logs` |
| The pipeline it runs | `app_get_configuration` | `get_current_configuration` |
| What each module did last | - | `get_module_data_by_id`, `get_metrics` |
| Its modules | - | `get_all_modules` |
| Packages, known vulnerabilities, licenses | `app_get_installed_packages`, `app_get_vulnerabilities`, `app_get_licenses` | - |

The app tools of the hub take the app's internal `id` (from `app_get_my`); `task_create` and
`data_query_execute` take its `app_id`. Start with `app_get_my` and filter, rather than reading
every Core.

## Reading the log

Modules log as `collectu.<module_name>.<id>`. What the common lines mean:

| Line | Meaning | Look at |
|---|---|---|
| `Could not start module '...' with the id '...'. Retrying in 10 seconds: ...` | The start raises - mostly a connection refused, a wrong address or wrong credentials. Retried until it works | The error after the colon; the module's parameters |
| `Could not start module ... Import of third party packages failed.` / `Could not import required packages` | A requirement of the module is missing on the Core | `auto_install` in the Core's settings; the module's `third_party_requirements` |
| `The module did not report to be ready within 10 s.` | A module's `start()` blocks without setting `self.started`; data waited | The module's code (a bug in the module) |
| `Worker for ... has been processing for ...s` | A module downstream is slow or blocked - it holds up its link | The module it names, and what it writes to |
| `Queue for ... is full (10000 data objects).` | An output or processor cannot keep up; **new data is dropped** | The destination of that module; whether it is buffered |
| `... did not stop within 3 s and are leaked` / `did not end within 3 s and is leaked` | A module does not end its threads on stop; they run until the application restarts | The module's code |
| `Received not permitted task with command: '...'` | The Core refuses a command not in its `allowed_commands` | The Core's `settings.ini` - the owner's decision |
| `Task signature verification failed` / `Refused to run task ... a second time` | A task was not signed by the hub, or replayed - not executed | The Core's clock; whether `HUB_JWKS_URL` is the hub it reports to |
| `Authentication with hub '...' failed` / `no valid HUB_API_ACCESS_TOKEN is set` | The Core's hub token is invalid or missing; it stops reporting | `hub_api_access_token` in the Core's `settings.ini` |
| `<secret-removed>` in an error | A configuration from a public source lacks a credential | The module's secret parameters |
| `Could not automatically search and download '...'` | A configured module is not installed and could not be downloaded | The module name and version; the owner's module policy |

A Core whose status is offline reports nothing: what the hub has is from before. Say how old it is
(the app's last report) rather than presenting it as current.

## Acting on a Core

Only when the user asks, for the Core they named, after confirming its `app_id`:

| Command (`task_create`) | Does |
|---|---|
| `restart` | Restarts the whole application |
| `stop` / `start` | Stops / starts the running pipeline again |
| `load` | Replaces the pipeline with the `configuration` sent, and starts it |
| `save` | Stores the `configuration` sent as a file on the Core |
| `update` | Updates the Core to its newest signed release, and restarts if anything changed |

A task reaches a Core with its next request to the hub - within seconds if it is online, otherwise
when it comes back, for at most a day. A Core runs only what its `allowed_commands` permit; a
refused command is the owner's decision, not something to work around. On a local Core,
`stop_module` and `start_module` restart a single module, and `stop_configuration` stops the
pipeline - all of them affect a running data flow, so ask first.

## Reporting

Lead with the answer (what is wrong, since when, which module), then the evidence - the log lines
that show it, quoted - then what the user can do. Do not paste whole logs.
