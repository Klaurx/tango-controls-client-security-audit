# Research Methodology and Approach

## How This Research Was Conducted

The analysis was performed entirely through manual static code reading of the
cppTango source tree. No dynamic analysis, fuzzing, instrumented builds, or
automated static analysis tools were used. The findings are based on reading
the source code directly and reasoning about reachable execution paths,
data flow from network inputs to sensitive operations, and invariant
violations in the logic.

The source tree was examined in the following order of priority, based on
an assessment of which components handle untrusted external data and which
enforce security-relevant decisions:

1. ZMQ event supplier and consumer (src/server/ZmqEventSupplier.cpp,
   src/client/ZmqEventConsumer.cpp)  these sit at the network boundary
   and process raw bytes from remote sources before any validation.

2. Poll ring buffer (src/server/PollRing.cpp)  shared data structure
   between the polling thread and client-facing reader threads, with
   complex type-dispatch logic across multiple IDL versions.

3. Access Control proxy (src/client/AccessProxy.cpp) — enforces the
   command-level security policy; logic bugs here have direct operational
   security consequences.

4. File database parser (src/client/FileDatabase.cpp)  parses operator-
   supplied configuration files; examined but no significant findings.

5. Readers-writers lock implementation
   (src/include/tango/internal/common/ReadersWritersLockInternal.h) 
   examined for concurrency correctness; the implementation is unusual
   (supports recursive writer-to-reader downgrade) but no exploitable
   race was confirmed.

6. Event keep-alive thread (src/client/EventKeepAliveThread.cpp) 
   examined for manual lock/unlock pairs that could leave locks held
   on exception; all exception paths were found to be covered by
   catch-all handlers.

## What Was Not Examined

The CORBA transport layer (omniORB internals) was not analyzed. The IDL
definitions were referenced for understanding data structures but not
audited for IDL-level vulnerabilities. The Python bindings (PyTango) were
not examined. The Tango database server (TangoDB) was not examined.

## Confidence Level

Each finding was validated through multiple passes of the relevant code.
For the High severity findings, the data flow from network input to the
vulnerable operation was traced end to end. The absence of length checks
was confirmed by reading both the calling code and the called functions.
For the Medium severity findings, the logic was traced across all call
sites identified by grep to confirm there was no compensating mechanism
elsewhere.

No finding is claimed without being able to point to the exact lines
of code where the bug manifests.
