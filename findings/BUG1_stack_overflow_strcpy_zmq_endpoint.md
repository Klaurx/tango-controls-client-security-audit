# BUG1: Stack Buffer Overflow via Unchecked strcpy of Server-Supplied ZMQ Endpoint Strings

**Severity:** High  
**File:** `src/client/ZmqEventConsumer.cpp`  
**Functions:** `connect_event_channel()`, `connect_event_system()`  
**Approximate lines:** 1598, 1607, 1613, 1965, 1981, 1984  

---

## Description

When a Tango client subscribes to events on a device server, it calls the
`ZmqEventSubscriptionChange` command on the server's admin device over CORBA.
The server returns a `DevVarLongStringArray` containing ZMQ endpoint strings
that the client will use to connect its subscriber sockets. These strings are
then copied into a fixed-size `char buffer[1024]` allocated on the stack using
`::strcpy` with no length validation whatsoever.

This occurs in two separate functions.

---

## Affected Code

### Site 1: connect_event_channel()

```
char buffer[1024];
int length = 0;

buffer[length] = ZMQ_CONNECT_HEARTBEAT;  // 1 byte
length++;

buffer[length] = 0;                       // 1 byte
length++;

// server-supplied string, no length check
::strcpy(&(buffer[length]), ev_svr_data->svalue[valid_endpoint << 1].in());
length = length + ::strlen(ev_svr_data->svalue[valid_endpoint << 1].in()) + 1;

// second server-supplied string built from channel name + ".heartbeat"
std::string sub(event_channel_name);
sub = sub + '.' + HEARTBEAT_EVENT_NAME;
::strcpy(&(buffer[length]), sub.c_str());
```

Two server-controlled strings are written sequentially into the same 1024-byte
stack buffer after 2 bytes of fixed header. The total available space before
overflow is 1022 bytes. Either string alone, or their combined length, can
exceed this.

### Site 2: connect_event_system()

```
char buffer[1024];
int length = 0;

// cmd byte + flag byte written first (2 bytes total)

// server-supplied endpoint string, no length check
::strcpy(&(buffer[length]), endpoint.c_str());
length = length + endpoint.size() + 1;

// server-supplied event name, no length check
::strcpy(&(buffer[length]), new_event_callback.received_from_admin.event_name.c_str());
```

Same pattern. Both `endpoint` and `event_name` originate from the server's
`ZmqEventSubscriptionChange` return value.

---

## Why There Is No Length Constraint

CORBA strings in omniORB are not length-limited below the GIOP message size
ceiling, which defaults to approximately 2 megabytes. The Tango IDL defines
these fields as plain `string` with no `maxLength` constraint. There is no
validation in the `ZmqEventSubscriptionChange` command handler on the server
side that limits what it returns, and there is no validation on the client
side before the `strcpy` call.

In a legitimate deployment, endpoint strings are of the form
`tcp://x.x.x.x:PORT` and are well under 1024 bytes. A rogue server faces no
technical obstacle to returning a string of arbitrary length.

---

## Trigger Conditions

A rogue or compromised device server returns from `ZmqEventSubscriptionChange`
with a `svalue` entry longer than 1022 bytes. The client calls
`subscribe_event()` on any attribute of that server, which triggers
`connect_event_channel()` and `connect_event_system()`, which perform the
unchecked `strcpy` calls.

The overflow occurs in the ZMQ event consumer setup path, which runs on the
thread calling `subscribe_event()`.

---

## Impact

The stack buffer for `buffer[1024]` is overwritten beyond its bounds. On
platforms without stack canaries the return address of the enclosing function
is overwritten, giving an attacker control of the instruction pointer when
the function returns. On platforms with stack canaries the canary mismatch
is detected and the process terminates.

In both cases the outcome is either process crash or, in the absence of
modern stack protections, arbitrary code execution in the context of the
client process.

---

## Threat Model Assessment

This requires a rogue or compromised server. Tango does not enforce
cryptographic server authentication by default. A client subscribing to
events on a server it does not fully trust is exposed to this bug. In
large control system deployments where clients subscribe to devices
operated by other groups or facilities, this is a realistic scenario.

---

## Recommended Fix

Replace the fixed stack buffer and `strcpy` calls with a `std::string`
or `std::vector<char>` based approach that grows dynamically, or perform
an explicit length check before each `strcpy` and throw an appropriate
exception if the total required length exceeds the buffer size.

The simplest safe fix is to build the message as a `std::string` and
pass it to the ZMQ message constructor directly, eliminating the fixed
buffer entirely.
