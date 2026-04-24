# Why No Proof of Concept Is Provided

This is my view on the decision not to include exploit proof-of-concept code
in this repository. It is not a policy statement. It is an honest account of
the reasoning.

---

## The Technical Reason

BUG1 and BUG2 both require a rogue device server. To build a PoC that
demonstrates either bug, you need:

1. A fake Tango admin device that responds to the `ZmqEventSubscriptionChange`
   CORBA command and returns a crafted `DevVarLongStringArray`.
2. A ZMQ PUB socket that publishes on the topic the client subscribed to,
   with the crafted payload (for BUG2).

Step 1 requires implementing CORBA server stubs for the Tango IDL. This is
not a trivial afternoon task. omniORB stub generation, server skeleton
registration, the ORB event loop, and the specific command handler for
`ZmqEventSubscriptionChange` all need to be wired together correctly before
a single packet is sent. Getting this wrong produces a server that the client
simply fails to connect to, not a server that exercises the bug.

The result is that a proper PoC for these two bugs would be a substantial
piece of software, not a short script. Writing it carefully enough to be
reproducible and not misleading would take considerable time.

---

## The Practical Reason

The bugs are self-evident from the source code. Any engineer who reads the
relevant lines can verify the issue in minutes without needing a running
demonstration. A PoC adds proof of exploitability beyond what static analysis
already establishes, but for the purposes of a bug report to the maintainers,
the code is the evidence.

BUG3 and BUG4 do not need a PoC at all. BUG3 is a copy-paste error that is
visible by inspection and reproducible by anyone with a polled device returning
DEV_USHORT or DEV_ULONG. BUG4 is a cache that never invalidates, confirmed
by a codebase-wide search showing exactly one insert and one find for the
relevant map.

---

## The Principled Reason

Tango Controls is deployed in scientific research facilities. The people
running these systems are not adversaries. A working rogue-server exploit
targeting the ZMQ event consumer is not a tool that benefits defenders in
this context. The maintainers need the code references, the reasoning, and
the fix recommendations. They do not need a ready-made attack.

I am not making a general statement that PoCs are always wrong to publish.
In contexts where vendors are unresponsive or findings are disputed, a PoC
is sometimes the only way to force acknowledgment. That is not the situation
here. The cppTango project is openly maintained and the bugs are clear enough
to stand on their own.

---

## What I Would Do Differently With More Time

If I were to invest the time to build a PoC, I would focus on BUG2 rather
than BUG1. BUG2 produces a deterministic crash (process termination) that
is immediately observable, making it easier to demonstrate reliably. The
alignment condition (shift_mem != 0) is met 75% of the time statistically,
but with control over the server's ZMQ socket configuration it may be
possible to influence the allocation alignment deterministically in a
controlled lab environment.

BUG1 is a stack overflow whose exploitability beyond crash depends on the
target platform's stack protection configuration. On a modern Linux system
with stack canaries and non-executable stacks, BUG1 produces a crash.
On an older or differently configured system it may be more serious. The
PoC complexity is the same in either case, but the demonstrated impact
varies significantly by target.

---

## Summary

No PoC is provided because the bugs are clear from static analysis, the
PoC implementation cost is disproportionate to the benefit for responsible
disclosure purposes, and a working rogue-server exploit is not the right
artifact for this audience and context.

The findings stand on the code.
