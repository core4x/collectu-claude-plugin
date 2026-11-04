---
name: pipeline-configuration
description: "Build, change, check or deploy a Collectu pipeline configuration - the list of modules (inputs, variables, tags, processors, outputs) and their links that a Collectu Core runs. Finds the modules on the Collectu Hub, fills in their parameters, validates the configuration, saves it on the hub and sends it to a Core, or starts it on a local Core. Use when the user wants data collected, transformed or stored by a Collectu Core, or asks about a configuration."
---

# Collectu pipeline configurations

A configuration is a JSON list of module objects (a Core keeps it as a YAML file). Each object
names its module and version, its parameters, and where its data goes:

```json
[
  {"id": "mqtt1", "module_name": "inputs.mqtt.client_1", "version": 25, "x": 216, "y": 72,
   "host": "192.168.0.10", "port": 1883},
  {"id": "mqtt1_var1", "module_name": "inputs.mqtt.client_1.variable", "version": 25, "x": 216, "y": 264,
   "input_module": "mqtt1", "topic": "plant/line1/#", "measurement": "line1", "links": ["filter1"]},
  {"id": "filter1", "module_name": "processors.filters.field_filter_1", "version": 4, "x": 600, "y": 264,
   "field_key": "temperature", "field_value": "*", "links": ["db1"]},
  {"id": "db1", "module_name": "outputs.databases.influxdb_2", "version": 9, "x": 1008, "y": 264,
   "host": "192.168.0.20", "bucket": "plant", "org": "plant", "token": "..."}
]
```

- `id` is unique in the configuration. `links` lists the ids a module sends its data to; never
  its own id. Inputs and outputs have no `links`.
- A variable or tag module (`<input module>.variable`, `.tag`) names its input module in
  `input_module`. The input module holds the connection; its children read through it.
- Tag modules enrich the data passing through: a module links to a tag module like to a
  processor.
- `x`, `y` place the module on the canvas of the editor - a module is 250x80, later modules of the
  pipeline further right with at least 50 between them.
- `active`, `name`, `description`, `panel` and `start_priority` are optional.
- A parameter with dynamic support takes `${local.key}` (a key of the data object passing
  through - tag, processor and output modules), `${<module id>.key}` (the latest data of another
  module) or `${env.KEY}` (an environment variable of the Core).

## Workflow

1. **Know what runs.** Changing a Core's pipeline starts from the configuration it runs
   (`app_get_configuration` on the hub, `get_current_configuration` on a local Core) or the one the
   user names (`configuration_search`, `configuration_get_my_by_id`). Amend it - keep ids,
   positions and parameters the user did not ask to change.
2. **Find the modules.** Infer the capabilities the request needs, then call
   `module_search_for_agents` with several short capability queries rather than the user's
   sentence: protocols, data formats, synonyms (`["opc ua client", "opcua subscription"]`). Search
   again with other terms if nothing fits. Prefer official, non-deprecated modules.
3. **Read every module you place.** `module_get_details_for_agents` gives the parameters with their
   types, defaults and whether they are required. Use the version it names - or, for a local Core,
   the version installed there (`get_all_modules`): another version is downloaded and replaces it.
4. **Fill in the parameters.** Take values from the request and the running configuration;
   otherwise the default, and say which defaults you relied on. A parameter marked
   `REQUIRED, NO DEFAULT` must come from the user: **never invent a host, IP address, port, topic,
   node id, credential, file path or database name, and no placeholders** (`your-...`,
   `example.com`, `changeme`, `<...>`). Ask for every missing value in one message, grouped by
   module, and wait.
5. **Validate.** On a local Core, `validate_configuration` (no side effects) reports errors and
   which modules starting it would download. The hub validates a configuration when it is saved -
   `valid` on the stored version. Fix every error before going on.
6. **Deliver it** the way the user asked - and only after they confirmed the target:

| Goal | How |
|---|---|
| Keep it on the hub | `configuration_create` (title of at most 50 characters) or `configuration_update` with a `commit_message`; `configuration_get_diff` shows what changed |
| Run it on a Core now | `task_create` with `command: "load"` and the `configuration` |
| Store it as a file on a Core | `task_create` with `command: "save"` |
| Run it on a local Core | `start_configuration` with `wait_seconds` (a few seconds), then read the states it answers and `get_logs` |

## Rules for production Cores

Cores run machines' data flows. Starting a configuration **replaces** the running pipeline: every
running module is stopped first.

- Confirm the target app before sending anything (`app_get_by_app_id`), and its status: a task for
  an offline Core waits until it reports again (for at most a day), and says so to the user.
- A Core executes only the commands its owner allowed (`allowed_commands` of the app on the hub and
  in the Core's `settings.ini`). A refused command is no reason to try another one.
- The module policy of the app's owner can refuse modules or versions; the error names them. Do not
  swap in other modules to get past it - tell the user.
- Secrets in a configuration are stored on the hub. A public configuration has them replaced by
  `<secret-removed>`, and a Core refuses to start a module that still has the placeholder.
- After deploying, check that it runs: the app's logs on the hub are as fresh as its last
  heartbeat (`app_get_logs`); a local Core answers at once (`get_logs`, `get_metrics`).

## When something is missing

A configuration needs a module nobody has written yet: say so, and offer to write one (the
`module-development` skill). The hub may also hold an article in the `Skill` category for the
task (`article_search`) - follow it if one applies, and say which.
