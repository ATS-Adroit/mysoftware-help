# adroitconfig-py — API Reference

This document describes every call programmers can make through the **adroitconfig-py** package: parameters, return types, exceptions, and examples.

The package wraps the Adroit Agent Server OLE Automation interface (ProgID `adroitserver`) for **configuration** tasks — creating agents, scanning tags, logging, alarming, and group membership. Runtime read/write methods (`Fetch`, `Poke`, etc.) are intentionally blocked by the wrapper.

---

## Requirements

| Requirement | Details |
|---|---|
| Platform | Windows only |
| Python | 3.10+ |
| Dependency | `pywin32>=305` |
| Adroit | Agent Server running with OLE Automation (MIS Server) installed |
| 64-bit Windows | Use **32-bit Python** (`SysWOW64`) — the Adroit OLE server is 32-bit |

Install:

```bash
python -m pip install adroitconfig-py
```

---

## Package exports

```python
from adroitconfig import AdroitConfigOLE, adroitconfig
```

| Name | Type | Description |
|---|---|---|
| `AdroitConfigOLE` | class | Main wrapper class. Creates and manages the COM connection. |
| `adroitconfig` | `AdroitConfigOLE` | Module-level singleton (`auto_connect=False`). Useful for scripts and REPL sessions. |

---

## Quick start

```python
from adroitconfig import AdroitConfigOLE

# Create wrapper and connect to local Agent Server
adroit = AdroitConfigOLE()

# Connect to a named server (remote or local)
adroit.Connect(None, "MyAgentServer")

# Create an analog agent
adroit.CreateAgent("ANA-01", "Tank level", "Analog")

# Scan rawValue from a PLC device
adroit.ScanTag("ANA-01", "rawValue", "PLC1", "DB1.DBD0", 1000, 0, 0)

# Enable historical logging
adroit.LogTag("ANA-01", "rawValue", 1, 24, 5, None)

# Clean up COM resources
adroit.close()
```

Using the module singleton:

```python
from adroitconfig import adroitconfig

adroitconfig.connect()
adroitconfig.Connect(None, "MyAgentServer")
# ...
adroitconfig.close()  # also called automatically at process exit
```

---

## `AdroitConfigOLE` — Python wrapper methods

These methods are implemented directly by the package (not proxied from COM).

### `AdroitConfigOLE(prog_id="adroitserver", auto_connect=True)`

Create a new wrapper instance.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `prog_id` | `str` | `"adroitserver"` | COM ProgID of the Adroit OLE server |
| `auto_connect` | `bool` | `True` | If `True`, call `connect()` immediately in `__init__` |

**Returns:** `AdroitConfigOLE` instance (not yet connected to an Agent Server — see `Connect()` below)

**Example:**

```python
# Lazy connection — connect manually later
adroit = AdroitConfigOLE(auto_connect=False)
adroit.connect()
```

---

### `connect() -> AdroitConfigOLE`

Initialize the COM apartment and dispatch the Adroit OLE server object. Safe to call multiple times; subsequent calls are no-ops if already connected.

**Returns:** `self` (the `AdroitConfigOLE` instance)

**Raises:** Any exception from COM initialization or dispatch failure (e.g. Adroit not installed, wrong Python bitness)

**Example:**

```python
adroit = AdroitConfigOLE(auto_connect=False)
adroit.connect()
```

---

### `close() -> None`

Release the COM object reference and uninitialize the COM apartment. Safe to call when not connected.

**Returns:** `None`

**Example:**

```python
adroit.close()
```

---

## COM configuration methods

Once connected (`connect()` has succeeded), the following methods are forwarded to the underlying Adroit OLE server.

Unless noted, **return type is `Any`** — the underlying COM method returns a variant whose exact shape depends on the Adroit server version and method. Treat the return value as an opaque success indicator unless your environment documents otherwise.

> **Licensing note:** `AlarmTag`, `LogTag`, and `ScanTag` perform internal pokes and consume **scan point licenses** per unique tag touched. See the [Adroit Installation Read Me](https://ats-adroit.github.io/mysoftware-help/Adroit_Installation_Read_Me.html).

---

### `Connect(Computer, Server) -> Any`

Connect the OLE wrapper to an Agent Server.

| Parameter | Type | Description |
|---|---|---|
| `Computer` | `str \| None` | Remote computer name. Use `None` or `""` for the local machine. |
| `Server` | `str \| None` | Agent Server name as configured in Adroit. |

**Returns:** `Any` — COM result object

**Raises:** `ConnectionError` — if the connection fails

**Example — local server:**

```python
adroit.Connect(None, "ProductionServer")
# or
adroit.Connect("", "ProductionServer")
```

**Example — remote server:**

```python
adroit.Connect("SCADA-HOST-01", "ProductionServer")
```

---

### `CreateAgent(Name, Description, Type) -> Any`

Create a new agent on the connected Agent Server.

| Parameter | Type | Description |
|---|---|---|
| `Name` | `str` | Unique agent name |
| `Description` | `str \| None` | Human-readable description |
| `Type` | `str \| None` | Agent type name (e.g. `"Analog"`, `"Digital"`, `"Device"`) |

**Returns:** `Any` — COM result object

**Raises:** `RuntimeError` — on failure (e.g. duplicate name, invalid type)

**Example:**

```python
adroit.CreateAgent("ANA-01", "Tank 1 level", "Analog")
adroit.CreateAgent("DIG-01", "Pump run feedback", "Digital")
adroit.CreateAgent("PUMP-01", None, "Digital")
```

---

### `RemoveAgent(Name) -> Any`

Remove an agent from the connected Agent Server.

| Parameter | Type | Description |
|---|---|---|
| `Name` | `str` | Agent name to remove |

**Returns:** `Any` — COM result object

**Example — remove a user agent:**

```python
adroit.RemoveAgent("ANA-01")
```

**Example — remove a logging agent** (see `LogTag`):

```python
# Removes logging configured with LogTag("ANA-01", "rawValue", ...)
adroit.RemoveAgent("LOG$ANA-01$rawValue")
```

---

### `ScanTag(Agent, Slot, Device, Address, Rate, Deadband, OutputEnable) -> Any`

Configure I/O scanning for an agent slot (typically `rawValue`).

| Parameter | Type | Description |
|---|---|---|
| `Agent` | `str` | Agent name, or `"agent.slot"` combined form |
| `Slot` | `str` | Slot name (ignored if `Agent` already contains a dot, e.g. `"ANA-01.rawValue"`) |
| `Device` | `str \| None` | Front-end device agent name |
| `Address` | `str \| None` | Protocol-specific scan address |
| `Rate` | `int` | Scan period in **milliseconds**. Use a **negative** value to unscan. |
| `Deadband` | `int` | Deadband in engineering units |
| `OutputEnable` | `int` | `1` = output (write-enabled) point; `0` = input |

**Returns:** `Any` — COM result object

**Example — scan an analog input at 1 s:**

```python
adroit.ScanTag("ANA-01", "rawValue", "NX7_PLC", "W0", 1000, 0, 0)
```

**Example — scan using combined agent.slot syntax:**

```python
adroit.ScanTag("ANA-01.rawValue", "", "NX7_PLC", "W0", 1000, 0, 0)
```

**Example — alias scan (internal reference):**

```python
adroit.ScanTag("REJ21", "rawValue", "ALIASCAN", "@REJ2.rawValue", 1000, 0, 0)
```

**Example — enable output (write) scanning:**

```python
adroit.ScanTag("DIG-01", "rawValue", "NX7_PLC", "R16.0", 100, 0, 1)
```

**Example — stop scanning (unscan):**

```python
adroit.ScanTag("ANA-01", "rawValue", None, None, -1, 0, 0)
```

---

### `LogTag(Agent, Slot, Set, Period, Rate, Filename) -> Any`

Configure historical data logging for an agent slot.

| Parameter | Type | Description |
|---|---|---|
| `Agent` | `str` | Agent name, or `"agent.slot"` combined form |
| `Slot` | `str` | Slot name (ignored if `Agent` contains a dot) |
| `Set` | `int` | Logging set number |
| `Period` | `int` | Log period in **hours** |
| `Rate` | `int` | Minimum logging interval in **seconds** |
| `Filename` | `str \| None` | Optional path to a datalog file |

**Returns:** `Any` — COM result object

**Example — log every 5 seconds, 24-hour period:**

```python
adroit.LogTag("ANA-01", "rawValue", 1, 24, 5, None)
```

**Example — log to a specific datalog file:**

```python
adroit.LogTag("ANA-01", "rawValue", 1, 168, 10, r"C:\Adroit\Logs\tank1.dlg")
```

**Removing logging:** use `RemoveAgent` with the synthetic logging agent name:

```python
adroit.RemoveAgent("LOG$ANA-01$rawValue")
```

---

### `AlarmTag(Agent, AlarmAgent, Type, Route, Ack, TypeEnum) -> Any`

Configure or remove alarming for an agent/tag.

| Parameter | Type | Description |
|---|---|---|
| `Agent` | `str` | Agent name, or `"agent.slot"` combined form |
| `AlarmAgent` | `str` | Alarm handler agent name |
| `Type` | `str` | Alarm type name. Leave empty when using `TypeEnum`. |
| `Route` | `int` | Alarm route number. Use **`-1`** to delete the alarm. |
| `Ack` | `int` | `1` = alarm must be acknowledgeable; `0` = not |
| `TypeEnum` | `int` | Numeric alarm type enum. Use **`-1`** when `Type` is provided. |

**Returns:** `Any` — COM result object

**Raises:** `RuntimeError` — on failure

**Example — configure alarm by type name:**

```python
adroit.AlarmTag("ANA-01", "ALM_MGR", "HiHi", 1, 1, -1)
```

**Example — configure alarm by enum:**

```python
adroit.AlarmTag("ANA-01.rawValue", "ALM_MGR", "", 1, 1, 3)
```

**Example — remove alarm:**

```python
adroit.AlarmTag("ANA-01", "ALM_MGR", "", -1, 0, -1)
```

---

### `Join(Group, MemberList) -> Any`

Add one or more agents to a group.

| Parameter | Type | Description |
|---|---|---|
| `Group` | `str` | Group name |
| `MemberList` | `str \| None` | Comma-separated list of agent names |

**Returns:** `Any` — COM result object

**Example:**

```python
adroit.Join("TANK_GROUP", "ANA-01,ANA-02,ANA-03")
```

---

### `Leave(Group, MemberList) -> Any`

Remove one or more agents from a group.

| Parameter | Type | Description |
|---|---|---|
| `Group` | `str` | Group name |
| `MemberList` | `str \| None` | Comma-separated list of agent names |

**Returns:** `Any` — COM result object

**Example:**

```python
adroit.Leave("TANK_GROUP", "ANA-01")
```

---

## Blocked COM methods

The wrapper **deliberately blocks** the following runtime I/O methods. Calling them raises `AttributeError`:

| Blocked method | Purpose |
|---|---|
| `Poke` | Write a tag value |
| `Fetch` | Read a single tag value |
| `FetchValues` | Read historically logged values |
| `FetchChanges` | Read historically logged value changes |
| `GetSlotInfo` | Query agent slot metadata |

**Example of blocked access:**

```python
>>> adroit.Fetch("ANA-01.rawValue")
AttributeError: 'AdroitConfigOLE' object has no attribute 'Fetch'
```

Use Adroit OPC, ActiveX (`AdroitX.ocx`), or other interfaces for runtime I/O. This package is scoped to configuration only.

---

## Complete workflow example

```python
from adroitconfig import AdroitConfigOLE

def configure_tank_level_tag():
    adroit = AdroitConfigOLE(auto_connect=False)
    try:
        adroit.connect()
        adroit.Connect(None, "PlantServer")

        # 1. Create agent
        adroit.CreateAgent("TANK-01-LVL", "Tank 1 level transmitter", "Analog")

        # 2. Scan from PLC
        adroit.ScanTag(
            "TANK-01-LVL", "rawValue",
            Device="S7_PLC",
            Address="DB10.DBD0",
            Rate=1000,
            Deadband=0,
            OutputEnable=0,
        )

        # 3. Historical logging (1 set, 7 days, min 10 s between samples)
        adroit.LogTag("TANK-01-LVL", "rawValue", Set=1, Period=168, Rate=10, Filename=None)

        # 4. Hi-Hi alarm on rawValue
        adroit.AlarmTag("TANK-01-LVL", "ALARM_MGR", "HiHi", Route=1, Ack=1, TypeEnum=-1)

        # 5. Add to operational group
        adroit.Join("TANKS", "TANK-01-LVL")

        print("Configuration complete.")
    finally:
        adroit.close()

if __name__ == "__main__":
    configure_tank_level_tag()
```

---

## Error handling

| Situation | Typical exception |
|---|---|
| COM object not connected | `AttributeError`: `"Adroit COM object is not connected"` |
| Blocked method called | `AttributeError`: `"'AdroitConfigOLE' object has no attribute '...'"` |
| Agent Server connection failure | `ConnectionError` (from `Connect`) |
| Agent creation / alarm failure | `RuntimeError` (from `CreateAgent`, `AlarmTag`) |
| Adroit not installed / wrong Python bitness | Generic exception from `connect()` |

Wrap production scripts with try/finally to ensure `close()` runs:

```python
adroit = AdroitConfigOLE()
try:
    adroit.Connect(None, "MyServer")
    # ... configuration calls ...
finally:
    adroit.close()
```

---

## Method summary

| Method | Purpose | Return type |
|---|---|---|
| `AdroitConfigOLE(...)` | Construct wrapper | `AdroitConfigOLE` |
| `connect()` | Initialize COM / dispatch OLE server | `AdroitConfigOLE` |
| `close()` | Release COM resources | `None` |
| `Connect(Computer, Server)` | Connect to Agent Server | `Any` |
| `CreateAgent(Name, Description, Type)` | Create agent | `Any` |
| `RemoveAgent(Name)` | Remove agent | `Any` |
| `ScanTag(...)` | Configure / remove scanning | `Any` |
| `LogTag(...)` | Configure historical logging | `Any` |
| `AlarmTag(...)` | Configure / remove alarming | `Any` |
| `Join(Group, MemberList)` | Add agents to group | `Any` |
| `Leave(Group, MemberList)` | Remove agents from group | `Any` |

