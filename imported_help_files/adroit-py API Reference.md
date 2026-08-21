# adroit-py API Reference

Python wrapper for the **Adroit OLE server** runtime on Windows. This document lists every callable surface exposed to application code: parameters, return types, and examples.

## Requirements

- **Python** 3.10+
- **Windows** with the Adroit OLE server installed and registered
- **pywin32** (`pip install adroit-py` installs it on Windows)

## Installation

```bash
python -m pip install adroit-py
```

For local development:

```bash
python -m pip install -e .
```

## Quick start

```python
from adroit import AdroitOLE, adroit

# Option A: create your own instance (connects immediately by default)
client = AdroitOLE()
value = client.Fetch("ANA-01", "rawValue")

# Option B: use the module-level singleton (does NOT connect on import)
adroit.connect()
values = adroit.FetchTags(["ANA-01.rawValue", "ANA-02.rawValue"])
adroit.close()
```

---

## Public exports

| Name        | Type        | Description                                      |
|-------------|-------------|--------------------------------------------------|
| `AdroitOLE` | class       | COM wrapper; create instances for your own lifecycle |
| `adroit`    | `AdroitOLE` | Module-level singleton (`auto_connect=False`)    |

Import:

```python
from adroit import AdroitOLE, adroit
```

---

## Tag naming convention

Most high-level helpers expect tags in **`agent.slot`** form (dot-separated):

| Form              | Example           | Meaning                          |
|-------------------|-------------------|----------------------------------|
| `agent.slot`      | `"ANA-01.rawValue"` | Agent `ANA-01`, slot `rawValue` |
| `agent`, `slot`   | `"ANA-01"`, `"rawValue"` | Same tag, two arguments     |

Invalid examples (raise `ValueError`):

```python
"rawValue"          # missing agent
"ANA-01"            # missing slot (when dotted form is required)
```

---

## Slot value types

When reading tags, `Fetch` / `FetchTags` return Python-native types depending on the Adroit slot type:

| Adroit type | Python type   |
|-------------|---------------|
| BOOLEAN     | `bool`        |
| INTEGER     | `int`         |
| REAL        | `float`       |
| STRING      | `str`         |
| TAGNAME     | `str`         |
| VARSTRING   | `str`         |
| TIME        | `datetime.datetime` |
| LIST        | `list`        |

Slot type numbers from `GetSlotInfo`:

| Code | Type      |
|------|-----------|
| 0    | BOOLEAN   |
| 1    | INTEGER   |
| 2    | REAL      |
| 3    | STRING    |
| 4    | TIME      |
| 5    | TAGNAME   |
| 6    | AGENTID   |
| 7    | RAWBYTES  |
| 8    | LIST      |
| 9    | AGENT     |
| 10   | VARSTRING |
| 11   | HEADER    |

---

## `AdroitOLE` — lifecycle

### `AdroitOLE(prog_id="adroitserver", auto_connect=True)`

Create a wrapper around the Adroit COM server.

| Parameter       | Type   | Default          | Description |
|-----------------|--------|------------------|-------------|
| `prog_id`       | `str`  | `"adroitserver"` | COM ProgID of the Adroit server |
| `auto_connect`  | `bool` | `True`           | Call `connect()` immediately |

**Returns:** `AdroitOLE` instance

**Example:**

```python
from adroit import AdroitOLE

# Connect on construction (default)
client = AdroitOLE()

# Defer connection until needed
client = AdroitOLE(auto_connect=False)
client.connect()
```

---

### `connect() -> AdroitOLE`

Initialize COM and attach to the Adroit server. Safe to call when already connected (no-op).

**Returns:** `self`

**Raises:** COM / connection exceptions if the server is unavailable

**Example:**

```python
client = AdroitOLE(auto_connect=False)
client.connect()
```

---

### `close() -> None`

Release the COM object and uninitialize COM. Safe to call when not connected (no-op).

The module-level `adroit` singleton registers `close()` with `atexit` so the process cleans up on exit.

**Example:**

```python
client = AdroitOLE()
try:
    print(client.Fetch("ANA-01", "rawValue"))
finally:
    client.close()
```

---

## Reading tags

### `Fetch(Agent: str, Slot: str) -> Any`

Read a single tag value from the connected server.

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `Agent`   | `str` | Agent name, or `"agent.slot"` (slot part of `Agent` is used; `Slot` is ignored) |
| `Slot`    | `str` | Slot name when `Agent` is only the agent name |

**Returns:** Value in a Python-native type (see [Slot value types](#slot-value-types))

**Example:**

```python
from adroit import AdroitOLE

client = AdroitOLE()

# Two-argument form
temp = client.Fetch("ANA-01", "rawValue")

# Dotted agent.slot in first argument
temp = client.Fetch("ANA-01.rawValue", "")
```

---

### `FetchTags(tag_names) -> dict[str, Any]`

Read multiple tags in one call. This is a **Python wrapper enhancement** over the raw COM `FetchTags`.

| Parameter   | Type              | Description |
|-------------|-------------------|-------------|
| `tag_names` | `list[str]` / `tuple[str]` | Tag names as `"agent.slot"` strings |

Also accepts `tag_names=` as a keyword argument.

**Returns:** `dict[str, Any]` mapping each input tag string to its value. On per-tag read failure, that entry is `None` (other tags still succeed).

**Raises:**

- `TypeError` if a tag name is not a `str`
- `ValueError` if a tag is not in `agent.slot` form

**Example:**

```python
from adroit import AdroitOLE

client = AdroitOLE()

values = client.FetchTags(["ANA-01.rawValue", "ANA-02.rawValue", "PUMP-01.value"])
# {"ANA-01.rawValue": 42.5, "ANA-02.rawValue": 38.1, "PUMP-01.value": True}

# Keyword form
values = client.FetchTags(tag_names=["ANA-01.rawValue"])
```

Positional list form:

```python
values = client.FetchTags(["A1.value", "A2.value"])
```

---

### `FetchChanges(AgentSlot: str, Start, End) -> list[list[Any]]`

Retrieve logged **changes** for a tag over a time range.

| Parameter   | Type | Description |
|-------------|------|-------------|
| `AgentSlot` | `str` | Tag as `"agent.slot"` |
| `Start`     | `datetime` or `str` | Range start |
| `End`       | `datetime` or `str` | Range end |

**Returns:** List of rows, each `[timestamp, value, integrity_flag]`:

| `integrity_flag` | Meaning        |
|------------------|----------------|
| 1                | NORMAL         |
| 4                | LAST LOGGED    |
| 5                | POST STARTUP   |
| 6                | BAD            |

**Example:**

```python
from datetime import datetime, timedelta
from adroit import AdroitOLE

client = AdroitOLE()
end = datetime.now()
start = end - timedelta(hours=24)

rows = client.FetchChanges("ANA-01.value", start.strftime("%Y-%m-%d %H:%M:%S"), end.strftime("%Y-%m-%d %H:%M:%S"))
for ts, value, flag in rows:
    print(ts, value, flag)
```

---

### `FetchValues(AgentSlot: str, Start, End, Samples) -> list[list[Any]]`

Retrieve **interpolated** logged values (evenly spaced samples).

| Parameter   | Type | Description |
|-------------|------|-------------|
| `AgentSlot` | `str` | Tag as `"agent.slot"` |
| `Start`     | `datetime` or `str` | Range start |
| `End`       | `datetime` or `str` | Range end |
| `Samples`   | `int` | Number of samples to return |

**Returns:** List of rows, each `[timestamp, interpolated_value, integrity_flag]`

**Example:**

```python
from datetime import datetime, timedelta
from adroit import AdroitOLE

client = AdroitOLE()
end = datetime.now()
start = end - timedelta(hours=1)

series = client.FetchValues("ANA-01.value", start.strftime("%Y-%m-%d %H:%M:%S"), end.strftime("%Y-%m-%d %H:%M:%S"), 60)
for ts, value, flag in series:
    print(ts, value, flag)
```

---

### `GetSlotInfo(Agent: str) -> list[list[Any]]`

List metadata for all slots on an agent.

| Parameter | Type  | Description   |
|-----------|-------|---------------|
| `Agent`   | `str` | Agent name    |

**Returns:** List of rows, each `[slot_name: str, description: str, type_no: int]`

**Example:**

```python
from adroit import AdroitOLE

client = AdroitOLE()
for slot_name, desc, type_no in client.GetSlotInfo("ANA-01"):
    print(slot_name, desc, type_no)
```

---

## Writing tags

### `SetTag(...) -> Any | dict[str, Any]`

Write one or more tag values. Wraps the underlying COM **`Poke`** method with convenient bulk and dotted-name syntax.

#### Single tag — three arguments

```python
SetTag(Agent: str, Slot: str, value: Any) -> Any
```

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `Agent`   | `str` | Agent name |
| `Slot`    | `str` | Slot name |
| `value`   | `Any` | Value to write |

**Returns:** COM result from `Poke`, or `None` on error

**Example:**

```python
from adroit import AdroitOLE

client = AdroitOLE()
client.SetTag("PUMP-01", "setpoint", 75.0)
```

#### Single tag — dotted name

```python
SetTag("agent.slot", value) -> Any
```

**Example:**

```python
client.SetTag("PUMP-01.setpoint", 75.0)
```

#### Multiple tags — dict

```python
SetTag({"agent.slot": value, ...}) -> dict[str, Any]
```

**Returns:** `dict[str, Any]` mapping each tag to the COM result (or `None` on error)

**Example:**

```python
results = client.SetTag({
    "PUMP-01.value": 75.0,
    "VALVE-02.value": True,
})
```

#### Multiple tags — list of pairs

```python
SetTag([("agent.slot", value), ...]) -> dict[str, Any]
```

**Example:**

```python
results = client.SetTag([
    ("PUMP-01.setpoint", 75.0),
    ("VALVE-02.value", True),
])
```

#### Multiple tags — keyword

```python
SetTag(tag_values={...}) -> dict[str, Any]
# or
SetTag(tag_values=[("agent.slot", value), ...]) -> dict[str, Any]
```

**Example:**

```python
results = client.SetTag(tag_values={"PUMP-01.setpoint": 80.0})
```

**Raises:**

- `TypeError` — unrecognized arguments, or non-string tag names
- `ValueError` — tag not in `agent.slot` form (for dotted / bulk forms)

---

### `Poke(Agent: str, Slot: str, value: Any) -> Any`

Low-level COM write (proxied from the server). Prefer **`SetTag`** for dotted names and bulk updates.

**Example:**

```python
client.Poke("PUMP-01", "setpoint", 75.0)
```

---

## Server connection

### `Connect(Computer: str | None, Server: str | None) -> Any`

Connect to an **Agent Server** (distinct from the initial OLE wrapper connection).

| Parameter  | Type            | Description |
|------------|-----------------|-------------|
| `Computer` | `str` or `None` | Remote computer name; `None` or empty for local |
| `Server`   | `str` or `None` | Agent Server name |

**Returns:** Connection result object (COM-specific)

**Raises:** `ConnectionError` if the connection fails

**Example:**

```python
from adroit import AdroitOLE

client = AdroitOLE()
result = client.Connect(None, "MyAgentServer")
```

---

## Other COM methods

Additional methods on the Adroit COM object are forwarded via attribute access when connected. For example, the interactive entry point uses:

### `ping() -> Any`

Health-check style call (exact return type depends on the COM server).

**Example:**

```python
from adroit import adroit

adroit.connect()
print(adroit.ping())
adroit.close()
```

Only call COM-forwarded methods **after** `connect()` (or after constructing `AdroitOLE()` with default `auto_connect=True`).

---

## Blocked COM methods

The following COM methods exist on the underlying server but are **intentionally blocked** on `AdroitOLE` (calling them raises `AttributeError`):

| Method        | Purpose (COM)                    |
|---------------|----------------------------------|
| `AlarmTag`    | Configure/remove alarms          |
| `CreateAgent` | Create a new agent               |
| `Join`        | Add agents to a group            |
| `Leave`       | Remove agents from a group       |
| `LogTag`      | Configure historical logging     |
| `RemoveAgent` | Remove an agent                  |
| `ScanTag`     | Configure tag scanning           |

Use the native Adroit tooling or unblocked COM access if you need these operations.

Reference signatures (not callable through this wrapper):

```python
# AlarmTag(Agent, AlarmAgent, Type, Route, Ack, TypeEnum)
# CreateAgent(Name, Description, Type)
# Join(Group, MemberList)
# Leave(Group, MemberList)
# LogTag(Agent, Slot, Set, Period, Rate, Filename)
# RemoveAgent(Name)
# ScanTag(Agent, Slot, Device, Address, Rate, Deadband, OutputEnable)
```

---

## Module-level `adroit` singleton

```python
from adroit import adroit
```

| Property        | Value |
|-----------------|-------|
| Type            | `AdroitOLE` |
| `auto_connect`  | `False` at import |
| Cleanup         | `adroit.close()` registered with `atexit` |

**Typical pattern:**

```python
from adroit import adroit

adroit.connect()
try:
    print(adroit.FetchTags(["ANA-01.rawValue"]))
finally:
    adroit.close()
```

---

## Error handling

| Situation | Behavior |
|-----------|----------|
| Method called before `connect()` | `AttributeError`: *"Adroit COM object is not connected"* |
| Blocked COM method | `AttributeError`: *"has no attribute 'MethodName'"* |
| Invalid tag format in `FetchTags` / `SetTag` | `ValueError` |
| Non-string tag in bulk helpers | `TypeError` |
| Single-tag read failure in `FetchTags` | That key set to `None`; others continue |
| Single-tag write failure in bulk `SetTag` | That key set to `None`; others continue |
| COM server unavailable on `connect()` | Original COM exception propagated |

---

## Complete example

```python
from datetime import datetime, timedelta
from adroit import AdroitOLE

def main():
    client = AdroitOLE()
    try:
        # Read one tag
        print("Single:", client.Fetch("ANA-01", "rawValue"))

        # Read many tags
        tags = client.FetchTags(["ANA-01.rawValue", "ANA-02.rawValue"])
        print("Bulk read:", tags)

        # Write tags
        client.SetTag("PUMP-01.setpoint", 72.5)
        client.SetTag({
            "VALVE-01.value": True,
            "VALVE-02.value": False,
        })

        # Agent metadata
        for slot, desc, typ in client.GetSlotInfo("ANA-01"):
            print(f"  {slot}: {desc} (type {typ})")

        # Historical data
        end = datetime.now()
        start = end - timedelta(hours=1)
        changes = client.FetchChanges("ANA-01.value", start.strftime("%Y-%m-%d %H:%M:%S"), end.strftime("%Y-%m-%d %H:%M:%S"))
        print(f"Changes in last hour: {len(changes)} rows")

    finally:
        client.close()

if __name__ == "__main__":
    main()
```

---

## API summary

| Callable | Parameters | Return type |
|----------|------------|-------------|
| `AdroitOLE(...)` | `prog_id: str`, `auto_connect: bool` | `AdroitOLE` |
| `connect()` | — | `AdroitOLE` |
| `close()` | — | `None` |
| `Fetch(Agent, Slot)` | `str`, `str` | `Any` |
| `FetchTags(tag_names)` | `list[str]` | `dict[str, Any]` |
| `FetchChanges(AgentSlot, Start, End)` | `str`, time, time | `list[list[Any]]` |
| `FetchValues(AgentSlot, Start, End, Samples)` | `str`, time, time, `int` | `list[list[Any]]` |
| `GetSlotInfo(Agent)` | `str` | `list[list[Any]]` |
| `SetTag(Agent, Slot, value)` | `str`, `str`, `Any` | `Any` |
| `SetTag("agent.slot", value)` | `str`, `Any` | `Any` |
| `SetTag({...})` / `SetTag([...])` | mapping or pairs | `dict[str, Any]` |
| `SetTag(tag_values=...)` | mapping or pairs | `dict[str, Any]` |
| `Poke(Agent, Slot, value)` | `str`, `str`, `Any` | `Any` |
| `Connect(Computer, Server)` | `str\|None`, `str\|None` | `Any` |
| `ping()` | — | `Any` (COM) |
