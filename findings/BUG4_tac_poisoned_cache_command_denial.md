# BUG4: TAC Allowed Commands Cache Permanently Poisoned on Transient Server Error

**Severity:** Medium  
**File:** `src/client/AccessProxy.cpp`  
**Functions:** `AccessProxy::is_command_allowed()`, `AccessProxy::get_allowed_commands()`  
**Approximate lines:** 287, 340, 361  

---

## Description

The Tango Access Control (TAC) system enforces per-class command restrictions
in addition to the coarser user/host access level check. For a given device
class, the TAC server returns the list of commands that are permitted regardless
of the user's general access level. This list is fetched by
`get_allowed_commands()` and cached in `allowed_cmd_table` for the lifetime
of the process.

The bug is that `get_allowed_commands()` inserts an empty vector into the
cache when it fails to retrieve the list from the TAC server, and
`is_command_allowed()` unconditionally uses the cached value on all subsequent
calls without any mechanism to detect or recover from a poisoned entry.

---

## Affected Code

### get_allowed_commands()

```
void AccessProxy::get_allowed_commands(const std::string &classname,
                                       std::vector<std::string> &allowed)
{
    try
    {
        DeviceData din, dout;
        din << classname;
        dout = command_inout("GetAllowedCommands", din);
        dout >> allowed;
    }
    catch(Tango::DevFailed &)
    {
        allowed.clear();   // any error produces an empty list
    }

    // empty list inserted into cache unconditionally
    allowed_cmd_table.insert(
        std::map<std::string, std::vector<std::string>>::value_type(classname, allowed)
    );
}
```

On any `DevFailed` exception (network error, TAC server restart, timeout,
connection refused), `allowed` is cleared and then inserted into
`allowed_cmd_table` as an empty vector associated with `classname`.

### is_command_allowed()

```
bool AccessProxy::is_command_allowed(std::string &classname, const std::string &cmd)
{
    std::map<std::string, std::vector<std::string>>::iterator pos;
    pos = allowed_cmd_table.find(classname);

    std::vector<std::string> allowed;
    if(pos == allowed_cmd_table.end())
    {
        get_allowed_commands(classname, allowed);  // only called on cache miss
    }
    else
    {
        allowed = pos->second;  // cache hit: returns whatever is stored, including empty
    }

    if(allowed.empty())
    {
        ret = false;  // empty list -> all commands denied
    }
    // ...
}
```

`get_allowed_commands` is only called when `classname` is absent from
`allowed_cmd_table`. Once an entry exists, even an empty one, the cache
hit branch is taken on every future call and the TAC server is never
queried again for that class.

### Cache invalidation

A search of the entire codebase confirms that `allowed_cmd_table` has
exactly one `insert` call (in `get_allowed_commands`) and one `find` call
(in `is_command_allowed`). There is no clear, erase, expiry, or
re-query logic anywhere.

---

## Trigger Conditions

1. A client process has TAC enabled and an `AccessProxy` instance is active.
2. The first invocation of `is_command_allowed` for a particular device class
   occurs while the TAC server is transiently unreachable. This includes:
   - TAC server restart (which takes a few seconds)
   - Network blip between client host and TAC server host
   - TAC server overload causing a command timeout
   - TAC server not yet started at process launch time
3. The TAC server recovers and becomes available again.

After step 2, all subsequent calls to `is_command_allowed` for that device
class return false, regardless of TAC server availability, for the lifetime
of the client process.

---

## Impact

Users who have `ACCESS_WRITE` rights but are subject to per-command
restrictions via the TAC system will find that all commands on affected
device classes are permanently denied after a transient TAC error. The
denial manifests as an `API_ReadOnlyMode` exception thrown from both the
synchronous `command_inout` path (Connection.cpp lines 1284, 1489) and
the asynchronous `command_inout_asynch` path (Connection_asyn.cpp line 82).

No error message indicates that the denial is due to a stale cache rather
than a genuine access control decision. Operators see the same exception
as they would for a real access denial. The only resolution is to restart
the client process.

In scientific control systems, where operators may need to issue commands
during time-critical operations and a process restart may not be immediately
feasible, this silent lockout has real operational consequences.

---

## What This Is Not

This is not a privilege escalation. A poisoned cache denies access; it does
not grant it. The impact is operational disruption rather than a security
boundary violation.

---

## Recommended Fix

Do not insert into `allowed_cmd_table` when the fetch fails:

```
catch(Tango::DevFailed &)
{
    allowed.clear();
    return;  // do not cache, allow retry on next call
}
```

This ensures that a transient error causes a retry on the next call rather
than a permanent cache entry. If performance requires caching, a TTL or
a flag distinguishing "empty because no commands allowed" from "empty
because fetch failed" should be introduced.
