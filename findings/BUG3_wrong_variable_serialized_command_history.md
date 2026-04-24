# BUG3: Wrong Variable Serialized for DEV_USHORT and DEV_ULONG in Command History

**Severity:** Medium  
**File:** `src/server/PollRing.cpp`  
**Function:** `PollRing::get_cmd_history(long n, Tango::DevCmdHistory_4 *ptr, Tango::CmdArgType &cmd_type)`  
**Approximate lines:** 1647 to 1659  

---

## Description

`get_cmd_history` builds a compressed command history record by iterating
over the poll ring buffer, accumulating data into type-specific arrays, and
then serializing those arrays into the returned `DevCmdHistory_4` structure.

The function declares separate accumulator variables for each supported type:

```
Tango::DevVarDoubleArray  *new_tmp_db  = nullptr;
Tango::DevVarUShortArray  *new_tmp_ush = nullptr;
Tango::DevVarULongArray   *new_tmp_ulg = nullptr;
// ... and others
```

The data gathering loop correctly populates `new_tmp_ush` for `DEV_USHORT`
and `DEVVAR_USHORTARRAY`, and `new_tmp_ulg` for `DEV_ULONG` and
`DEVVAR_ULONGARRAY`.

However, in the subsequent serialization block that runs on the last
iteration, the cases for these four types serialize `new_tmp_db`
(the double accumulator) instead of the correct variable:

```
case Tango::DEVVAR_USHORTARRAY:
case Tango::DEV_USHORT:
    if(new_tmp_db != nullptr)       // should be new_tmp_ush
    {
        ptr->value <<= new_tmp_db;  // should be new_tmp_ush
    }
    break;

case Tango::DEVVAR_ULONGARRAY:
case Tango::DEV_ULONG:
    if(new_tmp_db != nullptr)       // should be new_tmp_ulg
    {
        ptr->value <<= new_tmp_db;  // should be new_tmp_ulg
    }
    break;
```

The double cases immediately above use `new_tmp_db` correctly. The ushort
and ulong cases appear to have been written by copy-pasting the double case
and failing to update the variable name.

---

## Consequences

There are two distinct failure modes depending on whether `new_tmp_db` is
null at the time of serialization.

### Case 1: new_tmp_db is null

If the command type is `DEV_USHORT` or `DEV_ULONG`, `new_tmp_db` is never
allocated during the gathering loop. The null check prevents serialization.
The returned `DevCmdHistory_4` contains an empty value field. The correctly
populated `new_tmp_ush` or `new_tmp_ulg` is leaked.

The client receives a history record with no data and no error indication.

### Case 2: new_tmp_db is non-null

This can only occur if the same `get_cmd_history` call processes a command
type that uses `new_tmp_db` (i.e., `DEV_DOUBLE` or `DEVVAR_DOUBLEARRAY`)
alongside the ushort or ulong cases in a single invocation. In practice
`cmd_type` is a single fixed type per call, so this case does not arise
in normal usage. However if it did, double-typed data would be transmitted
under a ushort or ulong type code, producing a type mismatch on the
receiving end.

### Memory leak

In all cases where the command type is `DEV_USHORT`, `DEVVAR_USHORTARRAY`,
`DEV_ULONG`, or `DEVVAR_ULONGARRAY`, the correctly populated `new_tmp_ush`
or `new_tmp_ulg` buffer is allocated and filled but never serialized and
never freed. It is leaked on every `command_history()` call for polled
commands of these types.

---

## Affected Operations

Any client calling `command_history()` on a polled command whose declared
output type is `DEV_USHORT`, `DEVVAR_USHORTARRAY`, `DEV_ULONG`, or
`DEVVAR_ULONGARRAY` will receive incorrect results. No exception is raised
on either the server or client side. The corruption is silent.

---

## Trigger Conditions

No adversarial precondition is required. Any deployment that:

1. Has a device command with output type DEV_USHORT, DEVVAR_USHORTARRAY,
   DEV_ULONG, or DEVVAR_ULONGARRAY
2. Has that command configured for polling
3. Has a client requesting command history

will encounter this bug.

---

## Recommended Fix

Replace `new_tmp_db` with the correct variable in the serialization block:

```
case Tango::DEVVAR_USHORTARRAY:
case Tango::DEV_USHORT:
    if(new_tmp_ush != nullptr)
    {
        ptr->value <<= new_tmp_ush;
    }
    break;

case Tango::DEVVAR_ULONGARRAY:
case Tango::DEV_ULONG:
    if(new_tmp_ulg != nullptr)
    {
        ptr->value <<= new_tmp_ulg;
    }
    break;
```
