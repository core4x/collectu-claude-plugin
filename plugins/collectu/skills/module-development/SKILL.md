---
name: module-development
description: "Write, review or fix a Collectu module (input, variable, tag, processor or output) - in a module folder of your own in a Collectu Core, or in the modules repository. Covers the class contract, the Configuration dataclass and its metadata, the start/stop/_run lifecycle, sync vs. async methods and their event loops, the readiness gate (self.started), parent input modules, buffering, data requirements, dynamic variables, testing a module and sending it to the Collectu Hub. Use whenever a Collectu module - a Python file under inputs/, processors/ or outputs/ of a module folder - is created, changed or reviewed."
---

# Collectu module development

A module is a single Python file. The runtime (the **collectu-core** repository) discovers it,
validates its configuration, instantiates it and drives it. This skill describes the contract that
file must satisfy, how to test it and how it reaches the Collectu Hub (section 13).

A module lives in one of two places. Find out which before writing anything:

- **A module folder of your own** in a Collectu Core: `collectu-core/src/modules/<folder>/`, named
  by `custom_module_folder` in the Core's `settings.ini` (often a git repository added as a
  submodule there). It has the directories `inputs/`, `processors/` and `outputs/`, each with an
  `__init__.py`, like every directory below them.
- **The modules repository** (core4x/modules), where the official modules are developed: a
  `modules/` directory next to `testkit/`, `test.py` and `maintenance.toml`. Its own tests and CI
  apply there - read [references/modules-repository-tests.md](references/modules-repository-tests.md)
  before writing or changing one, and follow the repository's own instructions where they differ.

Look at existing modules before writing a new one: the Collectu Hub's MCP server finds them
(`module_search`, `module_search_for_agents`) and shows their code (`module_get_by_id`).

## 1. Pick the module type

There are five module classes. They are *not* interchangeable — pick by what the module does.

| Class | Role | Produces data? | Receives data? | `_run` |
|---|---|---|---|---|
| `InputModule` | Owns a **shared resource** (connection, session, client) for its variable/tag children. Has no `links` in its configuration model and therefore never forwards data itself. | no | no | none |
| `VariableModule` | **Produces** data on its own schedule. Drives itself in `start()` and pushes with `self._call_links(data)`. | yes | no | none |
| `TagModule` | **Enriches** a passing data object with extra fields/tags. | no | yes | `_run() -> dict` |
| `ProcessorModule` | **Transforms** a passing data object. | no | yes | `_run(data) -> Data` |
| `OutputModule` | **Sink.** Writes data somewhere. Has no `links`. | no | yes | `_run(data) -> None` |

Two shapes exist for producing data:

- **Standalone variable module** (no parent) — one file, one `VariableModule`, everything it needs
  is created in its own `start()`. Example: a timer, a random generator, a file reader.
- **Input module + children** (with parent) — one `InputModule` holding a connection, plus a
  `VariableModule` and/or `TagModule` in the *same file* that use it through
  `self.input_module_instance`. Use this whenever several children must share one connection.

## 2. Naming

`module_name` is the dotted path of the file below the module folder (`modules/` of the modules
repository, or your own folder - whose name is not part of it), plus a suffix for children:

| File | Class | `module_name` |
|---|---|---|
| `modules/inputs/mqtt/client_1.py` | `InputModule` | `inputs.mqtt.client_1` |
| `modules/inputs/mqtt/client_1.py` | `VariableModule` | `inputs.mqtt.client_1.variable` |
| `modules/inputs/mqtt/client_1.py` | `TagModule` | `inputs.mqtt.client_1.tag` |
| `modules/processors/general/foo_1.py` | `ProcessorModule` | `processors.general.foo_1` |
| `modules/outputs/databases/bar_1.py` | `OutputModule` | `outputs.databases.bar_1` |

Rules:

- The class names are fixed. `InputModule`, `VariableModule`, `TagModule`, `ProcessorModule`,
  `OutputModule` — nothing else is discovered.
- By convention the filename ends with `_<n>`, the module version (no test checks it). A
  **breaking** change means a new file (`client_2.py`), never an edit of the old one.
- **Never edit `__version__` by hand** — the hub sets it when the module is sent (section 13), and
  CI does in the modules repository.
- A module of your own folder with the `module_name` of an installed module replaces it on that
  Core (a warning is logged). Pick a name no official module has, unless replacing it is the point.

## 3. File anatomy

Module-level dunders are declared once and reused by every class in the file, so a file with an
input plus two children stays consistent.

```python
"""
Markdown documentation for this module. This docstring is published as the module's
documentation, so describe behaviour, required privileges and gotchas here.
"""
__version__: int = 1
"""The auto-generated version of the module."""
from dataclasses import dataclass, field
from typing import Any

# Internal imports.
import config
from models.validations import OneOf, Range, Regex
from modules.base.inputs.base import AbstractInputModule, AbstractVariableModule, AbstractTagModule, models

__public__: bool = True
__description__: str = "This module does..."
__author__: str = "Your Name"
__email__: str = "your.mail@address.de"
__deprecated__: bool = False
__third_party_requirements__: list[str] = []
```

`__public__ = True` publishes the module on the hub for everyone to read and use once it is sent;
keep a module with your own logic or your own systems in it at `False`.

Import the base classes from the matching package:

| Type | Import |
|---|---|
| input / variable / tag | `from modules.base.inputs.base import AbstractInputModule, AbstractVariableModule, AbstractTagModule, models` |
| processor | `from modules.base.processors.base import AbstractProcessorModule, models` |
| output | `from modules.base.outputs.base import AbstractOutputModule, models` |

### Required class attributes

```python
class OutputModule(AbstractOutputModule):
    version: int = __version__
    public: bool = __public__
    description: str = __description__
    author: str = __author__
    email: str = __email__
    deprecated: bool = __deprecated__
    third_party_requirements: list[str] = __third_party_requirements__
    field_requirements: list[str] = []   # processor + output only
    tag_requirements: list[str] = []     # processor + output only
    can_be_buffer: bool = False          # output only
```

`can_be_buffer = True` obliges the class to implement `store_buffer_data(module_id, data)` and
`get_buffer_data(module_id)`, both thread-safe — several output queue workers call them at once.
Either may be an `async def` (section 7). It then runs on the buffer module's own event loop, so it
uses the client its `start()` created, and the concurrent calls interleave at every `await` instead.
`get_buffer_data` returns `None` right away when nothing is buffered — it never waits for data, or
the calling queue worker hangs.

## 4. The Configuration dataclass

Declare it directly inside the module class, inheriting the matching base model.

```python
    @dataclass
    class Configuration(models.OutputModule):
        """The configuration model of the module."""
        host: str = field(
            metadata=dict(description="The host address of the database.",
                          category="basic",
                          required=False),
            default="127.0.0.1")
        password: str = field(
            metadata=dict(description="The password for the basic authentication.",
                          secret=True,
                          category="auth",
                          required=False),
            default=None)
        interval: float = field(
            metadata=dict(description="Interval in seconds. If 0, the module runs once.",
                          category="basic",
                          required=False,
                          dynamic=True,
                          validate=Range(min=0, max=86400, exclusive=False)),
            default=1)
```

Every field needs a `default` (or `default_factory`). The base models supply `id`, `module_name`,
`version`, `active`, `name`, `description`, `panel`, `x`, `y` and `start_priority`; variable, tag
and processor models add `links`, `forward_latest_data_only` and `worker_count_per_link`; variable
adds `measurement`; tag adds `is_tag`, `is_field`, `replace_existing`; output adds `buffered` and
`is_buffer`. Do not redeclare those.

Every field needs a `description` and a default (the static checks of the modules repository fail
it otherwise), and the configuration created from its defaults alone has to pass the validation
of collectu-core - the contract tests there create it so. A `required` field whose default is `None` then gets a stand-in value: the
first `OneOf` option, a string its `Regex` matches, the middle of a `Range`, `False`, `[]`, `{}` or
the string `"test"`.

### Metadata keys

Exactly six keys are read by the runtime. Anything else is ignored.

| Key | Default | Meaning |
|---|---|---|
| `description` | — | Shown in the UI and in generated docs. Always set it. |
| `category` | `"basic"` | UI grouping. Convention: `basic`, `advanced`, `auth`. |
| `required` | `False` | A `None` value fails validation with a message naming the field. |
| `validate` | — | `OneOf`, `Range`, `Regex` or `NestedListClassValidation`. |
| `secret` | `False` | Redacts the value in the UI, API and exports. Set it on passwords, tokens and keys (a convention, not checked by the tests). |
| `dynamic` | `False` | The value may contain dynamic markers, resolved with `self._dyn()`. |

### Validators

```python
validate=OneOf(["disable", "require", "verify-ca"])
validate=Range(min=0, max=86400, exclusive=False)
validate=Regex(regex="^[a-zA-Z_][a-zA-Z0-9_]{0,39}$",
               error="{input} is not a valid name. Please use letters, digits and '_'.")
```

For a list of nested objects, use `NestedListClassValidation` and rebuild the children in
`__post_init__`:

```python
@dataclass
class Field:
    key: str = field(metadata=dict(description="Field key.", required=True, dynamic=True),
                     default=None)
    value: str | float | int | bool | list = field(
        metadata=dict(description="Field value.", required=False, dynamic=True),
        default="Your input")

    def __post_init__(self):
        validate_module(self)


    @dataclass
    class Configuration(models.VariableModule):
        fields: list[Field] = field(
            metadata=dict(description="Field list.",
                          required=False,
                          dynamic=True,
                          validate=NestedListClassValidation(child_class=Field,
                                                             child_field_name="fields")),
            default_factory=list)

        def __post_init__(self):
            super().__post_init__()
            setattr(self, "fields", [Field(**entry) for entry in self.fields])
```

### Offering choices to the UI

```python
    @staticmethod
    def get_config_data(input_module_instance=None) -> dict[str, Any]:
        """
        Retrieve options for selected configuration parameters of this module.

        :param input_module_instance: The parent instance, for variable and tag modules.
        :returns: A dictionary containing the parameter as key and a list of options as value.
        """
        return {"level": ["debug", "info", "error", "warning", "critical"]}
```

It is called without a parent (`input_module_instance=None`, also for variable and tag modules) -
by an editor before the parent is configured, and by the contract tests - and expect a `dict` within 30 s. Return the static options, or empty lists, when there
is no parent, and never connect anywhere then.

## 5. Third-party packages

Never import them at file scope — the file is imported for discovery on machines where the
package is absent.

```python
    third_party_requirements: list[str] = ["psycopg2==2.9.13"]

    @classmethod
    def import_third_party_requirements(cls):
        """
        Import third-party requirements here and add them to third_party_requirements.
        """
        global psycopg2
        import psycopg2
        # Submodules are not pulled in by importing the package itself.
        import psycopg2.extras
```

Declare every package in `third_party_requirements`, pinned to an exact version with `==`. A Core
installs exactly these packages for the module, so a module that works only because another
package happens to be installed fails on the next Core.

- **Same version as the other modules.** A Core installs the requirements of all its modules into one
  environment, so two modules pinning one package differently fight over it. Pin the version the
  other modules pin (the hub's `module_get_by_id` shows the `third_party_requirements` of a
  module). In the modules repository a different version is reported as `inconsistent-requirements`, unless
  `[holds]` in `maintenance.toml` keeps the module below it on purpose. A version below the one in
  `[canonical]` of `maintenance.toml` is an error.
- **What collectu-core brings along** (`requests`, `PyYAML`, `cryptography`, `Markdown`,
  `packaging`, `tinydb`, `GitPython` and their dependencies, e.g. `urllib3` and `certifi`) needs no
  declaration and may be imported at file scope. Never declare another version of one of them:
  a core would install it over its own.
- **Installable everywhere CI tests.** The requirements have to install together with
  collectu-core's on Linux with the python versions the Cores run (3.11 and 3.13; in the modules
  repository `[python] supported` of `maintenance.toml`).
  A package without wheels for both has to be built there, and fails every test
  of the module if it needs a system library CI does not install (`tests.yml` installs
  `libpq-dev` only).
- Do not leave them open (`>=`) to get updates — in the modules repository `update.py` keeps the
  pins up to date.

## 6. Lifecycle and the readiness gate

This is the part most commonly got wrong. Read it fully.

`start()` runs in its **own thread** (`Start_<id>`) and is **retried** every
`config.RETRY_INTERVAL` seconds until it succeeds. Data can arrive long before that. The runtime
therefore does not execute module logic until the module reports that it is ready:

| Type | `_run` invoked from | Gated on |
|---|---|---|
| Output | queue worker | own `started`, checked per data object |
| Processor `thread_safe=True` | queue worker | own `started`, per data object |
| Processor `thread_safe=False` | `run()`, inline on the caller thread | own `started`, per data object |
| Tag | `run()`, inline on the caller thread | own `started`, per data object |
| Variable / Input | never — they drive themselves | — |

`self.started` is a `threading.Event` on every module. **It is set automatically when `start()`
returns**, and cleared again before every start attempt, when an attempt raises, and when the
module is stopped — so it always means "ready right now", never "was ready once". That yields
three rules:

### Rule 1 — a `start()` that returns needs no action

Establish the resource, then return. The runtime marks the module ready.

```python
    def start(self):
        """
        Start the module. Raise an exception if something went wrong.
        """
        self.client = SomeClient(self.configuration.host)
        self.client.connect()
        self.logger.info("Successfully connected to '{0}'.".format(self.configuration.host))
```

### Rule 2 — a `start()` that blocks must signal readiness itself

If `start()` runs a loop for the module's lifetime it never returns, so it must call
`self.started.set()` at the exact point the module becomes usable — after the resource its `_run`
depends on exists, and before the loop.

```python
    def start(self):
        with connect(self.configuration.url) as self.connection:
            self.logger.info("Successfully connected to '{0}'.".format(self.configuration.url))
            # This start method blocks for the lifetime of the module, so it reports
            # itself that the connection is established and data can be processed.
            self.started.set()
            while self.active:
                time.sleep(1)
```

Signal at the *right* point. If `_run` publishes through an object created in a connect callback,
signal in that callback after the object exists — not when the socket opens.

Forgetting this is not fatal: the first data object waits `config.START_TIMEOUT` (10 s), a warning
naming the module is logged, and processing continues without the guarantee.

### Rule 3 — `start()` must raise on failure, never swallow

The return value of `start()` is not evaluated. A `start()` that catches its exception and returns
normally — or returns `False` — is reported as a **successful** start: the module is marked ready
and **never retried**, and `_run` then runs against a resource that was never created.

```python
    def start(self):
        try:
            self.client = SomeClient(self.configuration.host)
            self.client.connect()
        except Exception as e:
            self.logger.error("Could not connect to '{0}': {1}"
                              .format(self.configuration.host, str(e)), exc_info=config.EXC_INFO)
            # Re-raise, otherwise the failed connection is treated as a successful start.
            raise
```

### `self.active` and `stop()`

`self.active` becomes `False` on shutdown. Every loop must test it (`while self.active:`) so the
module ends. `stop()` releases the resource and must tolerate a module that never started — a
core stops a module whose start keeps failing, and the contract tests of the modules repository
stop every module without starting it. So `__init__` only stores what it is given (`self.client = None`); connecting and
starting threads belong in `start()`.

**Never block longer than `config.STOP_TIMEOUT` (3 s) in an uninterruptible call.** Python cannot
kill a thread: a module still inside `time.sleep()` when the stop deadline expires is reported as
a leaked thread and keeps running until the process ends. This matters most in `_run`, which runs
on the *caller's* thread. Slice any configurable wait:

```python
    def _sleep(self, seconds: float):
        """
        Sleep, but return as soon as the module is stopped.

        :param seconds: The seconds to sleep.
        """
        deadline = time.monotonic() + seconds
        while self.active:
            remaining = deadline - time.monotonic()
            if remaining <= 0:
                return
            time.sleep(min(0.1, remaining))
```

```python
    def stop(self):
        """
        Stop the module.
        """
        try:
            if self.client is not None:
                self.client.close()
        except Exception:
            pass
```

### Start and stop order

Buffer → outputs → processors → inputs → tags → variables; stopped in reverse. Within a group,
higher `start_priority` starts first. Creation only *spawns* the start thread and does not wait,
which is exactly why the readiness gate exists.

## 7. Sync and async methods

`start`, `stop`, `_run` and the buffer methods `store_buffer_data` and `get_buffer_data` may each
be a plain `def` or an `async def`. The runtime detects which at call time and dispatches
accordingly, so they can be mixed freely. Choose async because the client library is
asyncio-based — not for speed: it does not change the threading model.

```python
    async def _run(self, data: models.Data) -> models.Data:
        data.fields["result"] = await fetch(data.fields["url"])
        return data
```

**How an async method is run.** Every module has **one event loop of its own**, running on a
dedicated thread (`Loop_<id>`). Whichever thread calls one of them — `Start_<id>`, `Stop_<id>`,
a link worker or a queue worker — the runtime hands the coroutine to that loop and the calling
thread waits for the result. Consequences:

- **One client for all methods.** Most asyncio objects (aiohttp `ClientSession`, asyncpg
  connections, asyncua clients, `asyncio.Lock`/`Queue`/`Event`, …) only work on the loop that
  created them. `start`, `_run` and `stop` share the loop, so create the client in `start`, use it
  in `_run` and close it in `stop`:

```python
    async def start(self):
        try:
            self.client = await create_client(self.configuration.host)
        except Exception:
            # start is retried, so what this attempt created is released before raising.
            await self._close()
            raise
        self.logger.info("Successfully connected to '{0}'.".format(self.configuration.host))

    async def _run(self, data: models.Data):
        await self.client.write(data.measurement, data.fields, data.tags, data.time)

    async def stop(self):
        await self._close()
```

- **The loop keeps running between calls.** Tasks created by an async `start()`
  (`asyncio.create_task`) and a client's own background work (keep-alives, reconnects,
  subscriptions, a server's sockets) keep running after `start()` returned. So an async `start()`
  that sets up a client or server **returns** (Rule 1) — it does not block to keep the loop alive.
  Keep a reference to every task you create (an unreferenced task can be garbage collected) and
  cancel it in `stop()`.
- **Children share the parent's loop.** Tag and variable modules run their async methods on the
  loop of their input module, so they `await` the parent's client directly — no stored
  `self.loop`, no `run_coroutine_threadsafe` from async code. Never cancel `asyncio.all_tasks()`
  either: the loop is shared, so that also cancels the module's own `stop()` and the calls of its
  children.
- **Same time budget.** The caller still waits: an async `_run` occupies the calling worker thread
  (or the queue worker) exactly like a sync one, data accumulates upstream while it runs, and
  `config.STOP_TIMEOUT` still applies. Async gives no concurrency per caller.
- **Never block the loop.** `time.sleep`, `requests` and other sync I/O inside an `async def`
  stall the loop — every coroutine of the module, *and of its parent and siblings*. Use
  `await asyncio.sleep(...)`, sliced against `self.active` like `_sleep` in section 6, and
  `await asyncio.to_thread(...)` for an unavoidable blocking call.
- **The rules of section 6 are unchanged.** A blocking async `start()` (e.g. a variable module's
  polling loop) calls `self.started.set()`; a failing one raises.

**Lifetime.** The loop opens with the module's first async call and closes once the module is
stopped — an input module's loop once it and every child that used it are stopped. Calls still
running get `config.STOP_TIMEOUT` to finish; then everything left, tasks included, is cancelled.
A module restarted after that gets a fresh loop, and its `start()` creates its clients again.

**From sync code into the loop.** Code that does not run on the loop — a sync `_run`, the static
`get_config_data` (called from an API thread), the callback of a sync library — runs a coroutine on
a module's loop with `_invoke(async_function, *args)`. It waits without a timeout, so bound the
coroutine itself where needed:

```python
    @staticmethod
    def get_config_data(input_module_instance=None) -> dict[str, Any]:
        if input_module_instance is None:
            return {"node_id": []}

        async def browse() -> list[str]:
            return await asyncio.wait_for(_browse(input_module_instance.client), timeout=BROWSE_TIMEOUT)

        # Called from an API thread, so the browsing is handed to the event loop of the input module.
        return {"node_id": input_module_instance._invoke(browse)}
```

Never wait for the loop from code that already runs on it (`_invoke`, or
`run_coroutine_threadsafe(...).result()`): it would wait for itself forever. `_invoke` raises a
`RuntimeError` there instead — `await` the coroutine.

**No loops of your own.** `asyncio.new_event_loop()` + `run_until_complete`, or `asyncio.run()`
per call, puts the module's clients on a loop that the runtime does not keep running, and
`asyncio.run()` closes its loop after every call (connection pools then fail with *"Event loop is
closed"*). Make the methods `async` instead.

## 8. Parent input modules

A variable or tag module reaches its parent through `self.input_module_instance`. The runtime
**waits for the parent's `started` before starting the child**, so the parent's resource is
guaranteed to exist by the time the child's `start()` or `_run()` runs. Do not write your own
wait loop or `is None` guard for it.

Declare the dependency in the child's configuration:

```python
        input_module: str = field(
            metadata=dict(description="The id of the input module. The required input module has to be "
                                      "module_name: inputs.mqtt.client_1",
                          required=True),
            default=None)
```

If the parent loses its connection later, the child is **not** re-gated: its `_run` raises, the
error is logged, and the data object is dropped (links are not called). That is intended — it
recovers by itself once the parent reconnects.

An async client of the parent is used directly by the child's async methods: they run on the
parent's event loop (section 7).

## 9. `_run` per type

### Output

```python
    def _run(self, data: models.Data):
        """
        Method called when new data has to be processed.

        :param data: The data object to be processed.
        """
        try:
            self.client.write(data.measurement, data.fields, data.tags, data.time)
        except Exception as e:
            self.logger.error("Could not store data: {0}".format(str(e)), exc_info=config.EXC_INFO)
            # Connection problem -> retry later.
            self._buffer(data=data, invalid=False)
```

`self._buffer(data, invalid)` hands the object to the configured buffer module and **returns
`False` when the module is not `buffered` or no buffer module is configured — the data is then
lost.** Check the return value when that matters, and say so in the log.

An async `_run` uses `await self._buffer_async(data=data, invalid=False)` instead — same arguments,
same return value. A plain `self._buffer` there blocks the module's event loop while the buffer
module stores the data.

- `invalid=False` — the destination failed. Replayed once it is reachable again.
- `invalid=True` — the *data* is unusable (bad format, illegal identifier). Written to the `_bin`
  key and never retried. Never use this for connection errors, and never for data you want back.

### Processor

`_run` returns the data object. Returning `None` drops it.

```python
    def __init__(self, configuration: Configuration):
        super().__init__(configuration=configuration, thread_safe=False)

    def _run(self, data: models.Data) -> models.Data:
        data.fields["extra"] = 1
        return data
```

`thread_safe=False` (the default) runs `_run` inline on the calling thread — fastest, but the
implementation must be re-entrant. `thread_safe=True` gives the module its own queue and a single
worker thread, like an output module; use it when `_run` touches shared state or a client that is
not thread-safe. Do **not** call `_call_links` from `_run` — the caller forwards the return value.

### Tag

`_run` takes no data argument and returns the key/value pairs to merge. The current object is
available as `self.current_input_data`.

```python
    def _run(self) -> dict[str, Any]:
        """
        Method generates/requests the data for this module and returns it.

        :returns: A dict containing the generated key-value pairs.
        """
        return {"level": self._dyn(self.configuration.level)}
```

### Variable

No `_run`. Produce in `start()` and push with `_call_links`. Honour `interval == 0` as "run once".

```python
    def start(self):
        while self.active:
            start_time = time.monotonic()
            try:
                data = models.Data(measurement=self.configuration.measurement,
                                   fields={self._dyn(self.configuration.key): self._read()})
                self._call_links(data)
            except Exception as e:
                self.logger.error("Could not generate data: {0}".format(str(e)),
                                  exc_info=config.EXC_INFO)
            finally:
                if self.configuration.interval == 0:
                    break
                elapsed = time.monotonic() - start_time
                if self.configuration.interval > elapsed:
                    time.sleep(self.configuration.interval - elapsed)
```

Measure with `time.monotonic()` and subtract the elapsed time so the schedule does not drift.

## 10. The data object

```python
models.Data(measurement="my_measurement",   # str, required for the object to be forwarded
            fields={"temperature": 21.5},   # dict[str, Any] - the values
            tags={"machine": "m1"},          # dict[str, Any] - the metadata
            time=datetime.now(timezone.utc)) # datetime, defaults to now
```

A data object with an empty `measurement` is silently ignored by the runtime.

## 11. Data requirements

`field_requirements` and `tag_requirements` are validated before `_run` on **processor and output**
modules. A requirement is a chain of `(...)` expressions joined by `and`; several entries in the
list form an `or`. Empty is `[]`.

```python
    field_requirements: list[str] = ["(key * with ints/floats)"]
    tag_requirements: list[str] = ["(keys == 1) and (value event == \"init\")"]
```

Keywords: `key` (named or `*`, negate with `!`), `keys` (count), `length` (list length), `value`,
`with` (type), `and` / `or`, and `()`. Types: `int`, `float`, `number`, `str`, `bool`, `list` and
the list forms `ints`, `floats`, `numbers`, `strs`, `bools`; combine with `/`.

## 12. Dynamic variables

Any field with `dynamic=True` must be resolved through `self._dyn()` at use time — never read the
raw attribute.

```python
from modules.base.base import DynamicVariableException

try:
    key = self._dyn(self.configuration.key, "str")
except DynamicVariableException as e:
    self.logger.error("Could not resolve the key: {0}".format(str(e)), exc_info=config.EXC_INFO)
    return
```

Resolve inside `_run`/the loop, not in `__init__` — the value depends on the current data object.

## 13. Testing and sending a module to the hub

In the modules repository, follow
[references/modules-repository-tests.md](references/modules-repository-tests.md) instead: its test
kit, its layers and its CI decide there. For a module of your own folder:

### 13.1 Is it discovered?

A Core imports its modules when it starts - a new or changed file takes a restart. List what it
registered, from the root of collectu-core:

```
python src/main.py -m processors          # or inputs, outputs, all
```

The module has to appear with its `module_name` and description. A file that does not import is
missing there and logged as `Could not import and register custom module '...'` with the reason.

### 13.2 A test of its own

A module is a class; test it as one, with `unittest` and nothing else - from `collectu-core/src`,
where the imports of a module resolve. Create it from its `Configuration` as a Core does, start it,
drive `_run`, and stop it in `finally`:

```python
import unittest

import models
from modules.my_modules.processors.math.scale_1 import ProcessorModule


class TestScale(unittest.TestCase):
    def test_scales_the_field(self):
        module = ProcessorModule(configuration=ProcessorModule.Configuration(
            module_name="processors.math.scale_1", key="a", factor=3))
        try:
            module.start()
            result = module._run(models.Data(measurement="m", fields={"a": 2}))
        finally:
            module.stop()
        self.assertEqual(result.fields, {"a": 6})

    def test_resolves_a_dynamic_key(self):
        data = models.Data(measurement="m", fields={"name": "b", "b": 5})
        module = ProcessorModule(configuration=ProcessorModule.Configuration(
            module_name="processors.math.scale_1", key="${local.name}", factor=2))
        module.current_input_data = data  # What a Core sets before _run, for ${local...}.
        try:
            module.start()
            self.assertEqual(module._run(data).fields["b"], 10)
        finally:
            module.stop()
```

```
cd src && python -m unittest modules.my_modules.tests.test_scale_1
```

Cover what the module promises: every class of the file, a start that fails (against a port
nothing listens on - `start()` has to raise), what an output does when its destination fails
(`_buffer` returns `False` without a buffer module), `interval=0` of a variable module, and data
`_run` cannot handle. Serve a protocol inside the test (`http.server`, `socketserver`) rather than
relying on a device, and wait for what is expected instead of sleeping.

### 13.3 On a running Core

With the MCP server of a local Core connected (the `collectu-core` plugin), the Core runs the
module as it will in production:

| Step | Tool |
|---|---|
| Registered, in the version you expect | `get_all_modules` |
| Its configuration passes the validation | `validate_configuration` (no side effects) |
| A processor turns a data object into the expected one | `execute_processor_module` |
| An input or output works in a pipeline | `start_configuration` with `wait_seconds`, then `get_logs`, `get_metrics`, `get_module_data_by_id` |

`start_configuration` **replaces the pipeline the Core runs.** Use it only on a Core that is there
to experiment with, and ask before doing so on any other: read what runs there first
(`get_current_configuration`), and start that again afterwards.

### 13.4 Sending it to the hub

From the root of collectu-core, with `hub_api_access_token` set in `settings.ini`:

```
python src/main.py --send_modules processors.math.scale_1   # without a name: every module of your folder
```

Only the modules of your own folder are sent. The hub stores the module as a new version, sets
`__version__` and inspects the code. Other Cores receive it with `--update_modules`, or download it
when a configuration names it (`auto_download`). An owner whose module policy allows only approved
modules has to approve the new version before a configuration with it is accepted - an approval
binds to the exact code, so every new version needs one. The hub's `module_get_diff` shows what
changed between two versions.

## 14. Logging

`self.logger` is provided. Use `exc_info=config.EXC_INFO` on error logs, and `.format()` rather
than f-strings for log messages, matching the surrounding code.

| Level | Use for |
|---|---|
| `info` | successful connect, table created, policy applied |
| `warning` | degraded but working; something the user should fix |
| `error` | this data object or batch failed |
| `critical` | the module cannot work at all |

## 15. Checklist

- [ ] Class named exactly `InputModule` / `VariableModule` / `TagModule` / `ProcessorModule` / `OutputModule`
- [ ] `__version__` untouched; a breaking change means a new file
- [ ] Module docstring describes behavior, privileges and gotchas
- [ ] All class attributes set from the module-level dunders
- [ ] Every configuration field has `description` and a default; the defaults pass the validation
- [ ] `dynamic=True` fields are read through `self._dyn()`
- [ ] Third-party packages imported only in `import_third_party_requirements`, declared, pinned with
      `==` at the version the other modules pin, installable with python 3.11 and 3.13 on Linux
- [ ] `get_config_data` returns a dict without a parent module, without connecting anywhere
- [ ] `__init__` neither connects nor starts threads
- [ ] `start()` raises on failure — never returns `False`, never swallows
- [ ] A blocking `start()` calls `self.started.set()` at the point the module becomes usable
- [ ] Every loop tests `self.active`
- [ ] `stop()` is safe when `start()` never ran, and ends every thread the module started
- [ ] Output `_run` buffers with the right `invalid` flag; an async `_run` awaits `_buffer_async`
- [ ] Processor `_run` returns the data object and does not call `_call_links`
- [ ] No hand-written waiting for `input_module_instance` — the runtime guarantees it
- [ ] Errors logged with `exc_info=config.EXC_INFO`
- [ ] Async methods: no sync blocking calls inside; no event loop of their own, no hand-over between loops
- [ ] An async `start()` that sets up a client returns; its tasks are referenced and cancelled in `stop()`
- [ ] A test covers every class of the file, the failing start, and buffering, order or
      `interval=0` where they apply, and waits for results instead of sleeping
- [ ] The Core lists the module (`python src/main.py -m all`), and its configuration passes
      `validate_configuration`
- [ ] In the modules repository: `python test.py --module <name>` passes - no static error or
      warning, no failed test, nothing added to `tests/baseline.json`

## 16. Mistakes seen in review

| Mistake | Consequence |
|---|---|
| Blocking `start()` without `self.started.set()` | First data object stalls `config.START_TIMEOUT` (10 s), warning logged |
| `start()` catches its exception and returns | Failed start reported as success; never retried |
| `except SomeError:` without `as e`, body formats `str(e)` | `NameError` on every failure, hiding the real error |
| Signalling readiness at connect when `_run` needs a later object | `_run` runs against a `None` attribute |
| Two `connect()`/`run()` calls copied between an input and output variant | Second call is meaningless or reconnects needlessly |
| Swallowing a failed connect and entering the idle loop anyway | Module idles forever with no connection and is never retried |
| Own `while not ready: sleep()` guard for the parent | Dead code — the runtime already waits |
| Ignoring the return value of `_buffer` | Silent data loss when the module is not `buffered` or no buffer module is configured |
| `time.sleep(<configurable>)` in `_run` | Caller's thread outlives `STOP_TIMEOUT` and is reported leaked |
| A guard clause that only returns when `self.active` | Falls through into code that assumes the other branch |
| Async `start()` blocking only to keep the loop alive (`await event.wait()`, an idle `while self.active`) | Dead weight — the loop runs anyway; `stop()` then has to release it |
| Own loop (`new_event_loop()`, `asyncio.run()` per call) or a stored `self.loop` hand-over | Clients on a loop nobody keeps running; *"Event loop is closed"* after `asyncio.run()` |
| Cancelling `asyncio.all_tasks()` when a module ends | Also cancels its own `stop()` and its children's calls — the loop is shared |
| `_invoke` / `.result()` on the module's loop from code running on it | `RuntimeError` instead of a deadlock |
| Task from `asyncio.create_task(...)` not stored or not cancelled in `stop()` | May be garbage collected mid-run; otherwise runs until the loop is closed |
| `time.sleep` / sync I/O inside an `async def` | Stalls the shared loop — the module, its parent and its siblings |
| `self._buffer(...)` in an async `_run` | Blocks the module's loop while the buffer module stores — `await self._buffer_async(...)` |
| Connecting or starting a thread in `__init__` | Creating the module fails without the device; the contract tests report the thread as leaked |
| `stop()` using an attribute only `start()` sets | `AttributeError` when a core stops a module whose start keeps failing |
| `get_config_data` connecting without a parent | The editor waits on a connection; the contract tests fail after 30 s |
| A package pinned differently than in the other modules | The modules fight over one environment on a Core; in the modules repository `inconsistent-requirements` |
| A test with a fixed file name, topic or database | Collides with another test on the shared service or folder |
| `time.sleep(...)` in a test, then asserting | Passes locally, fails on a slow CI runner |
