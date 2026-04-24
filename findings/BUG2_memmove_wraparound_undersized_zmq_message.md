# BUG2: Unsigned Integer Wraparound Leading to Near-Unbounded memmove on Undersized ZMQ Event Message

**Severity:** High  
**File:** `src/client/ZmqEventConsumer.cpp`  
**Function:** `push_zmq_event()`  
**Approximate lines:** 2379 to 2410  

---

## Description

When a ZMQ event message is received and dispatched through `push_zmq_event()`,
the code performs a buffer alignment fixup before handing the raw bytes to the
CDR unmarshaller. The purpose of this fixup is to ensure that 64-bit data types
within the buffer are aligned on 8-byte boundaries, as required by omniORB's
unmarshalling routines.

The fixup computes the misalignment of the received buffer pointer and, if
misaligned, performs an in-place `memmove` to shift the data. The amount to
move is computed as `data_size - 4` using `size_t` arithmetic. If `data_size`
is less than 4 at this point, the subtraction wraps around to approximately
`SIZE_MAX`, and `memmove` is called with a near-maximum byte count.

There is no minimum size check on `data_size` before this arithmetic.

---

## Affected Code

```
char *data_ptr = (char *) event_data.data();
size_t data_size = event_data.size();          // attacker-controlled

bool shift_zmq420 = false;
int shift_mem = reinterpret_cast<std::uintptr_t>(data_ptr) & 0x3;

if(shift_mem != 0)
{
    char *src = data_ptr + 4;
    size_t size_to_move = data_size - 4;       // wraps if data_size < 4

    if(data_type == PIPE)
    {
        src = src + 4;
        size_to_move = size_to_move - 4;       // wraps again if data_size < 8
    }

    char *dest = src - shift_mem;
    if((reinterpret_cast<std::uintptr_t>(dest) & 0x7) == 4)
    {
        dest = dest - 4;
    }

    memmove((void *) dest, (void *) src, size_to_move);  // called with ~SIZE_MAX
}
```

The alignment condition `shift_mem != 0` is satisfied when the low 2 bits of
the buffer pointer are nonzero, which occurs for 3 out of 4 possible pointer
alignments. ZMQ's internal allocator does not guarantee any particular alignment.

For non-pipe events, the wraparound triggers when `data_size < 4`.
For pipe events, it triggers when `data_size < 8`.

---

## Why the Vulnerable Path Is Reachable

A ZMQ SUB socket filters messages by topic prefix. The client subscribes to
a specific event name string set by the server during the
`ZmqEventSubscriptionChange` negotiation. Only messages whose first frame
matches the subscribed topic are delivered to `push_zmq_event()`.

The four-frame ZMQ message structure expected is:
1. Event name (topic filter frame)
2. Endianness byte
3. Call info (CDR-marshalled ZmqCallInfo)
4. Event data payload

A rogue server controls all four frames. After passing the topic filter
(frame 1), it can send a fourth frame (the event data payload) of any size,
including 1, 2, or 3 bytes. The code in `push_zmq_event()` receives this
payload as `event_data` and passes `event_data.size()` directly into the
arithmetic above without any lower bound check.

---

## Why There Is No Try/Catch Around This Block

The `try` block that catches malformed data exceptions begins further down
in the function, around the CDR unmarshalling calls. The alignment fixup
block runs before any exception handling is in place. A crash in `memmove`
cannot be caught by C++ exception handling in any case.

---

## Impact

`memmove` called with `size_to_move` equal to approximately `SIZE_MAX`
(for example, `0xFFFFFFFFFFFFFFFC` on a 64-bit system) will attempt to
read from `src` and write to `dest` for a near-maximum number of bytes.
In practice this causes an immediate segmentation fault as the memory
access moves outside the process's mapped address space. The process
terminates.

Depending on the heap layout at the time of the call, there is a small
window before the segfault where heap metadata or adjacent allocations
are overwritten. Whether this is exploitable beyond a crash depends on
allocator internals and is not claimed here.

The reliable, guaranteed outcome is process crash on the ZMQ receive thread,
which terminates the entire client process and kills all active event
subscriptions.

---

## Threat Model Assessment

Same as BUG1. This requires a rogue or compromised server that the client
has subscribed to. A legitimate server always sends at minimum 8 bytes in
the event data frame (2 times CORBA::ULong of padding are prepended to
every legitimate payload). Only a deliberately crafted message triggers
the wraparound.

---

## Recommended Fix

Add a minimum size guard immediately before the alignment block:

```
if(data_size < 4)
{
    fill_deverror_for_malformed_event_data(ev_name, errors);
    // proceed to callback dispatch with error, skip unmarshalling
}
```

For pipe events the guard should be `data_size < 8`. This check costs
nothing in the normal path and completely eliminates the wraparound.
