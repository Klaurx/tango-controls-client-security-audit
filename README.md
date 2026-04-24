# cppTango Vulnerability Research

**Target:** cppTango (https://github.com/tango-controls/cppTango)  
**Codebase revision examined:** local clone of the main development branch, April 2025  
**Research method:** Manual static analysis  
**Findings:** 4 confirmed bugs (2 High severity, 2 Medium severity)  
**PoC status:** Not provided. See methodology note below.

---

## Repository Structure

```
tango-controls-client-security-audit/
    LICENSE
    README.md
    findings/
        BUG1_stack_overflow_strcpy_zmq_endpoint.md
        BUG2_memmove_wraparound_undersized_zmq_message.md
        BUG3_wrong_variable_serialized_command_history.md
        BUG4_tac_poisoned_cache_command_denial.md
    methodology/
        APPROACH.md
        WHY_NO_POC.md
```

---

## Summary of Findings

| ID   | Title                                               | File                        | Severity |
|------|-----------------------------------------------------|-----------------------------|----------|
| BUG1 | Stack buffer overflow via strcpy of server endpoint | ZmqEventConsumer.cpp        | High     |
| BUG2 | memmove near SIZE_MAX via undersized ZMQ message    | ZmqEventConsumer.cpp        | High     |
| BUG3 | Wrong variable serialized in command history        | PollRing.cpp                | Medium   |
| BUG4 | TAC allowed commands cache poisoned on error        | AccessProxy.cpp             | Medium   |

---

## Scope and Target

cppTango is the C++ implementation of the Tango Controls framework, a widely deployed
open-source distributed control system used in large-scale scientific facilities including
particle accelerators, telescopes, synchrotrons, and fusion experiments. Device servers
built on cppTango expose hardware controls over a network using CORBA for RPC and ZeroMQ
for the event system.

The research focused on the following surfaces, in order of priority:

1. The ZMQ event consumer and supplier (network boundary, untrusted data handling)
2. The poll ring buffer (shared memory between polling thread and reader threads)
3. The Access Control proxy (security enforcement logic)
4. The file database parser (local file parsing)

The CORBA transport layer itself was not analyzed. The focus was on cppTango's own
code above the CORBA abstraction.

---

## Threat Model

All High severity findings share the same threat model: a rogue or compromised device
server. This is a realistic threat in Tango deployments because:

- Tango does not enforce cryptographic authentication between clients and servers
  by default. The Access Control layer (TAC) controls which users can invoke commands
  but does not verify the identity of servers.
- Operators routinely connect clients to device servers operated by other teams,
  other institutes, or external facilities.
- A compromised device server anywhere in a control system can target any client
  that subscribes to its events.

The Medium severity findings are triggerable in normal operation with no adversarial
precondition.

---

## Disclosure Note

These findings were produced through original static analysis of the publicly available
cppTango source code. This repository constitutes public disclosure. The author
encourages the cppTango maintainers to review these findings and address them
through their standard development process.

---

## Copyright and Attribution

All findings, analysis, and methodology in this repository are the original work
of the author. See LICENSE for terms of use. Do not reproduce, republish, or claim
these findings as your own work.

**Author:** Klaurx the divine, 

Slowdive -- Ballad of the dead, 

Mt 
