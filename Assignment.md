# Assignment: Event-Driven Message Injection in a Mesh-Like Network

## I. Overview

In Parts 1–3, your Pico W nodes injected messages periodically and forwarded them using a flooding-based, mesh-like approach with de-duplication and TTL control.

In this assignment, you will extend the system so that **message injection is triggered by network events**, rather than time. This mirrors real IoT systems where local application logic reacts to observed network conditions.

---

## II. The Task

Modify your final `mesh_node.py` so that **receiving messages from the network triggers a new message injection**.

### 1. Functional Requirements

* **Trigger Condition:**
  When a node receives **5 new messages** from other nodes.

* **Definition of “New”:**
  A message is considered new only if it:

  * passes de-duplication (`RX NEW`)
  * does not originate from the node itself

* **Action:**
  Upon receiving the 5th new message, the node must:

  1. Display the details of all 5 received messages.
  2. Inject a new message of type **`R`**.

* **Message Type:**

  * `TYPE = 'R'`
  * `TTL = 0`

---

### 2. Behavioural Constraints

* `R` messages **must not be forwarded**.
* `R` messages **must not be counted** toward the “5 messages” trigger.
* After injecting an `R` message, the node must **reset its counter** and begin counting again.

---

## III. Implementation Guide

### A. Tracking Received Messages

You will need to maintain a small buffer or list that stores information about each **newly received** message.

* Only add a message to this buffer **after** it passes:

  * de-duplication
  * self-origin filtering
* Each entry should store at least:

  * origin node ID
  * message ID

---

### B. Triggering the Response

When the buffer reaches **5 entries**:

1. Print all stored message details to the console.
2. Construct and inject an `R` message summarising the received packets.
3. Clear the buffer so counting can begin again.

*Tip:* This logic should be placed **inside the receive path**, not in the main loop.

---

### C. Controlling Propagation

To ensure the response remains local and does not overload the network:

* Set `TTL = 0` for `R` messages.
* Explicitly prevent forwarding of messages with `TYPE = 'R'`.

---

## IV. Expected Output & Demo

You must demonstrate the behaviour using the serial console.

### Example Console Output

```
RX NEW: orig=A msgid=...
RX NEW: orig=B msgid=...
RX NEW: orig=C msgid=...
RX NEW: orig=D msgid=...
RX NEW: orig=E msgid=...

--- RECEIVED 5 PACKETS ---
1) orig=A msgid=...
2) orig=B msgid=...
3) orig=C msgid=...
4) orig=D msgid=...
5) orig=E msgid=...

INJECT (R): M1|X|...|0|R|A,B,C,D,E
```

---

## V. Learning Outcomes

By completing this task, you should be able to:

* Distinguish **time-based** vs **event-driven** message injection
* Apply de-duplication and TTL correctly in application logic
* Design network-aware application behaviour on top of a mesh-like system
* Reason about how protocol rules affect system-level behaviour

---

## VI. Grading Rubric

| Criteria                      | Points                                                                      |
| ----------------------------- | --------------------------------------------------------------------------- |
| **Correct Trigger Logic**     | `R` message is injected only after 5 new messages are received.             |
| **Accurate Filtering**        | Duplicates, self-origin messages, and `R` messages are not counted.         |
| **Correct Message Injection** | `R` message uses `TYPE='R'` and `TTL=0`.                                    |
| **Proper Reset Behaviour**    | Receive buffer is cleared after each injection.                             |
| **Clear Console Output**      | The 5 received messages and the injected `R` message are clearly displayed. |

---
