# Tests in the modules repository

Every module is tested on its own: `python test.py` installs exactly its
`third_party_requirements` and collectu-core's into an environment of its own, and runs the tests
against the newest commit of collectu-core (`maintenance.toml`). CI (`.github/workflows/tests.yml`)
runs them for every changed module on Linux with python 3.11 and 3.13, and a push to `main` sends
the modules to the hub only if they pass. The failures listed in `tests/baseline.json` are known;
any other one fails the run. A new module has no entry there, so **every failure of a new module
fails CI**.

| Layer | What | Where | Written for each module |
|---|---|---|---|
| 0 | Static checks of the source, no installation | `testkit/static.py` | no |
| 1 | The contract every module fulfils | `tests/test_contract.py` | no |
| 2 | The module's own tests, with nothing outside the test | `tests/<type>/<path>/test_<file>.py` | **yes** |
| 3 | The module's own tests against a service in Docker | same file, `service` fixture | **yes** |

Layers 0 and 1 need no test code. A module passing only those is known to import and to be
created, nothing more. **A new module ships a test file (layer 2, or 3 where it needs a database or
a broker), and a change of a module's behaviour comes with a test of it.**

## 1. Layer 0: static checks

| Check | Severity | Reports |
|---|---|---|
| `syntax` | error | a file that does not parse |
| `module-classes` | error | no class collectu-core discovers, or one not deriving from its `Abstract...` base |
| `configuration-fields` | error | a `Configuration` field without `field(metadata=dict(description=...))` or without a default |
| `file-scope-imports` | error | a third-party import outside `import_third_party_requirements` (section 5 of SKILL.md) |
| `requirement-format` | error | a requirement pip cannot parse |
| `outdated-requirements` | error | a pin below `[canonical]` of `maintenance.toml` (above it: a warning) |
| `start-readiness` | warning | a `start()` (not of a variable module) looping `while self.active:` without `self.started.set()` |
| `start-raises` | warning | a `start()` whose whole body is a `try` with a handler that catches everything and does not raise |
| `inconsistent-requirements` | warning | a package pinned differently than elsewhere in the repository |
| `unpinned-requirements` | warning | a requirement without `==` |

Errors fail CI. Keep a module free of warnings, too.

## 2. Layer 1: the contract

Runs for every module, in its environment, without any test code:

| Test | Passes if |
|---|---|
| `test_import` | the file imports and defines a class collectu-core discovers |
| `test_third_party_requirements` | every requirement is installed, and `import_third_party_requirements` imports them |
| `test_configuration` | the configuration of every class, created from its defaults (section 4 of SKILL.md), passes the validation |
| `test_config_data` | `get_config_data` returns a `dict` within 30 s; variable and tag modules get `input_module_instance=None` |
| `test_create_and_stop` | every class is created as on a core (children with their input module) and stopped **without having been started**, and no thread it started is left running |
| `test_processor_test_data` | the cases of `get_test_data` of a processor pass (section 5 below) |

So `__init__` must not connect or start threads, `stop()` must work without `start()` (section 6 of SKILL.md),
and `get_config_data` must answer without a parent (section 4 of SKILL.md).

## 3. Layers 2 and 3: the module's test file

The tests of `modules/<type>/<path>/<file>.py` are in `tests/<type>/<path>/test_<file>.py`, one
file for every class of the module file: `tests/inputs/mqtt/test_client_1.py` tests
`inputs.mqtt.client_1` and its `.variable` and `.tag`. `test.py` does not run a test file anywhere
else, or named differently.

```python
"""
Tests of outputs.web.rest_request_1 (layer 2), against an HTTP server of the standard library.
"""
import pytest

from models import Data
from testkit import harness, servers


@pytest.fixture
def http():
    with servers.http_server() as server:
        yield server


def _start(server, **values):
    return harness.start("outputs.web.rest_request_1", host=server.url, port=server.port, path="/data", **values)


def test_posts_the_data_object_as_json(http):
    with _start(http) as output:
        output.store(Data(measurement="m", fields={"a": 1}))
    request = http.requests.wait_for(1)[0]
    assert (request.method, request.path) == ("POST", "/data")


def test_buffers_data_if_the_server_is_not_reachable():
    # Sent again once the server is reachable, instead of being thrown away as invalid.
    with harness.start("outputs.web.rest_request_1", host="http://127.0.0.1", port=harness.free_port(),
                       path="/data", buffered=True) as output:
        output.store(Data(measurement="m", fields={"a": 1}))
        assert [data.fields for data in output.buffer.buffered(output.id)] == [{"a": 1}]
        assert output.buffer.binned(output.id) == []
```

`harness.start(name, **configuration)` creates the module as a core does, runs `start()` on a
thread of its own and waits until `self.started` is set (not for a variable module, whose `start()`
usually runs for its lifetime). A failed start is raised as `AssertionError` ("The start method
raised ...") instead of retried; for a variable module, a test failing afterwards names the error
of its start in a note. Leaving the `with` block stops the module as a core does and fails
the test if one of its threads is still running: a non-daemon thread after `config.STOP_TIMEOUT`
(3 s), a daemon thread after 15 s. Only the values the test is about are given; required fields get
the stand-ins of section 4 of SKILL.md.

| To drive | Use | Notes |
|---|---|---|
| a processor | `module.process(data)` | returns what `_run` returns |
| an output | `module.store(data)` | calls `_run` |
| a tag | `module.tag(data)` | returns the key-value pairs `_run` adds |
| a variable | `module.inbox.wait_for(n)` | the first n data objects it forwarded |
| a child | `harness.start("inputs.x.y_1.variable", parent=input_module)` | the input module started before |
| the way of a core | `module.send(data)` | `run`: requirement validation, an output's queue, errors logged instead of raised; what is forwarded arrives in `module.inbox` |
| a buffer module | `harness.start(..., buffered=True)` | `module.buffer.buffered(module.id)`, `module.buffer.binned(module.id)` |

`process`, `store` and `tag` set `current_input_data` as a core does (so `${local.key}` resolves),
raise what `_run` raises and give up after 10 s (`timeout=` for slow work). Further helpers:
`harness.free_port()`, `harness.data_path(...)` (the `data` folder relative paths of a module end up
in), `harness.wait_for(predicate)` (returns a bool) and `harness.assert_data_equal(actual, expected)`.

`parent`, `buffered`, `wait` and `timeout` are parameters of `harness.start` itself. For a module
with a configuration field of one of these names, use `harness.create(name, timeout=1).start()`:
`harness.start(name, timeout=1)` only shortens the wait for the start and leaves the module at its
default.

### What the tests cover

1. **Every class of the file** doing its job: the input module together with each of its variable
   and tag modules, with the defaults and with each option that changes the behaviour.
2. **A failing start raises** — for every module that connects to or opens anything: against a
   port nothing listens on, a missing file or wrong credentials.

   ```python
   def test_start_raises_without_server():
       with pytest.raises(AssertionError, match="start method raised"):
           harness.start("inputs.opcua.client_2", host="opc.tcp://127.0.0.1", port=harness.free_port(), timeout=60)
   ```

3. **Buffering** of an output: a destination that fails or is unreachable ends in `buffered`,
   data the destination rejects as unusable in `binned`.
4. **Variable modules**: what they produce and in which order, and `interval=0` producing exactly
   one data object.
5. **Dynamic fields**: a `dynamic=True` field set to `${local.<key>}` resolves against the data
   object.
6. **Bad input**: a data object `_run` cannot handle is dropped (`process` returns `None`), raises
   or is binned, as the module intends.

### Writing them

- **Nothing outside the test.** Serve the protocol inside the test with `testkit/servers.py`:
  `opcua_server`, `modbus_server`, `websocket_server`, `http_server`, `smtp_server`,
  `imap_server` and `ads_server` (pyads, Linux only). A protocol none of them speaks gets a small
  server in the test (`socketserver`), or a new one in `servers.py` once a second module needs it.
- **Docker (layer 3) only for databases and brokers.** Ask for one with the `service` fixture,
  which skips the test where Docker is not available:

  ```python
  def test_stores_a_row(service):
      postgres = service("postgres")   # .host, .port, .username, .password, .database, .extra
  ```

  There are `mqtt`, `postgres` (with TimescaleDB), `redis`, `mongodb` and `influxdb` (2.x). A new
  service goes into `testkit/docker-compose.yml`, published on `127.0.0.1` on a port of its own
  and with a health check, and into `ENDPOINTS` of `testkit/services.py`.
- **Names of their own.** A service runs for the whole run and is shared with the tests of other
  environments running at the same time, so each test uses a database, table, topic or key of its
  own (`uuid.uuid4().hex`). A module resolves a relative file path against the `data` folder
  (`harness.data_path("<module>", "<unique name>")`); `tmp_path` only works for a module that
  accepts an absolute path.
- **No timing assumptions.** Wait for what is expected (`inbox.wait_for(n)`,
  `server.requests.wait_for(n)`, `harness.wait_for(lambda: ...)`) instead of sleeping and then
  asserting. Sleep only to show that something does *not* happen, and bound durations generously
  (`0.4 <= delay < 3`): CI runners are slow.
- **Short timeouts in the module's configuration** (intervals, connect and read timeouts), so a
  failure path takes about a second.
- **Always a `with` block**: only leaving it checks for leaked threads. A module the test leaves
  running is stopped afterwards without that check.
- **Third-party packages** of the module may be imported at the top of the test file: it only
  runs in the module's environment.
- **A platform without a library** skips with a reason (`pytest.skip(...)`, as `ads_server` does
  on Windows) instead of failing.
- **Logs** are checked with `caplog`; a module logs as `collectu.<module_name>.<id>`.

## 4. Running them

```
python test.py --static                               # layer 0 of every module, in seconds
python test.py --module outputs.web.rest_request_1    # layers 0 to 3 of one module
python test.py --layer 2,3 --module outputs.web       # only the test files, of every module below outputs.web
python test.py --changed                              # the modules changed since origin/main
```

`--changed` tests every module after a change of the test kit or of `[core]` in
`maintenance.toml`. Layer 3 is skipped without Docker; CI runs it. The report is `.testrun/report.md`, the output of
pytest per environment is in `.testrun/groups/`. To debug a single test inside its environment
(`bin/python` on Linux):

```
.testenvs/<environment>/Scripts/python -m pytest tests/outputs/web/test_rest_request_1.py --collectu-modules outputs.web.rest_request_1
```

`python test.py --update-baseline` records the failures of a run as known ones. Use it after fixing
known failures — never to get a new module or a new failure through CI.

## 5. Processor `get_test_data`

Optional, and no replacement for a test file. Layer 1 creates the processor from the
`module_config` of each case, calls `start()`, then `_run(input_data)`, compares measurement, fields
and tags (not the time) with `output_data`, and stops it. `current_input_data` is set to the
`input_data` as on a core, so the configuration of a case can use `${local...}`; requirement
validation is skipped.

```python
    @classmethod
    def get_test_data(cls) -> list[dict]:
        test_1 = {
            "module_config": cls.Configuration(
                module_name=os.path.splitext(os.path.basename(__file__))[0], level="info"),
            "input_data": models.Data(measurement="test", fields={"test3": 1}, tags={"test1": 1}),
            "output_data": models.Data(measurement="test", fields={"test3": 1}, tags={"test1": 1}),
        }
        return [test_1]
```
