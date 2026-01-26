## BLE Advertisement Flooding “Mesh-like” Demo (MicroPython Pico W)

### Important note

MicroPython on Pico W supports BLE **advertising + scanning**, but it **does not provide a full BLE Mesh stack** (routing/provisioning/self-healing). 
This lab builds a **mesh-like flooding relay** using advertisements.

---

## Message format (frame)

We encode a small text frame inside the BLE “device name”:

```
M1|ORIG|MSGID|TTL|TYPE|DATA
```

* `ORIG`: original sender ID (we use `machine.unique_id()` as a stable per-board identifier)
* `MSGID`: unique per message (created by the origin)
* `TTL`: hop limit (used in Part 3)
* `TYPE`: e.g., `T` for temperature
* `DATA`: payload

---

# Starter template (students copy this first)

Create `mesh_node.py` with the template below. Then add snippets in Part 1 → Part 2 → Part 3.

```python
# mesh_node.py (TEMPLATE)
import bluetooth
import time
import ubinascii
import machine
import urandom
from micropython import const

# --------- BLE IRQ EVENTS ----------
_IRQ_SCAN_RESULT = const(5)
_IRQ_SCAN_DONE   = const(6)

# --------- CONFIG ----------
ADV_INTERVAL_US = 300_000      # 300 ms advertising interval
SCAN_MS         = 10_000       # scan window (restart continuously)

INJECT_PERIOD_S = 60           # originate new data once per minute
INJECT_JITTER_S = 10           # 0..10s jitter (prevents everyone injecting together)

DEFAULT_TTL     = 3            # used only in Part 3
SEEN_MAX        = 400          # used only in Part 3

# Stable per-board ID (acts like "MAC-like ID" for this lab)
NODE_ID = ubinascii.hexlify(machine.unique_id()).decode()[-6:]  # e.g. a1b2c3

# Pico internal temperature sensor (ADC4)
temp_adc = machine.ADC(4)
conv = 3.3 / 65535

def read_temp_c():
    v = temp_adc.read_u16() * conv
    return 27 - (v - 0.706) / 0.001721

def make_frame(orig, msgid, ttl, typ, data):
    return "M1|{}|{}|{}|{}|{}".format(orig, msgid, ttl, typ, data)

def parse_frame(s):
    try:
        if not s.startswith("M1|"):
            return None
        parts = s.split("|", 5)
        if len(parts) != 6:
            return None
        _, orig, msgid, ttl_s, typ, data = parts
        ttl = int(ttl_s)
        return orig, msgid, ttl, typ, data
    except:
        return None

def adv_payload_name(name_str):
    name = name_str.encode()
    payload = bytearray(b"\x02\x01\x06")                 # Flags
    payload += bytearray((len(name) + 1, 0x09)) + name   # Complete Local Name
    return payload

def frame_to_name(frame):
    # keep short so it fits in advertising data reliably
    return frame[:25]

class Node:
    def __init__(self):
        self.ble = bluetooth.BLE()
        self.ble.active(True)
        self.ble.irq(self._irq)

        # Part 3 uses this for de-duplication
        self.seen = []  # list of "orig:msgid"

        # Injection schedule (60s + jitter)
        self.next_inject_ms = time.ticks_add(
            time.ticks_ms(),
            (INJECT_PERIOD_S + self._rand_jitter_s()) * 1000
        )

        self.scan()

        print("Node ID:", NODE_ID)

    def _rand_jitter_s(self):
        return urandom.getrandbits(8) % (INJECT_JITTER_S + 1)

    def advertise_frame(self, frame):
        name = frame_to_name(frame)
        payload = adv_payload_name(name)
        self.ble.gap_advertise(ADV_INTERVAL_US, adv_data=payload)

    def scan(self):
        self.ble.gap_scan(SCAN_MS, 30000, 30000)

    # ----------------------------
    # PART-SPECIFIC FUNCTIONS GO HERE
    # ----------------------------

    def _irq(self, event, data):
        # PART-SPECIFIC IRQ HANDLING GOES HERE
        if event == _IRQ_SCAN_DONE:
            self.scan()

    def run(self):
        # PART-SPECIFIC MAIN LOOP GOES HERE
        while True:
            time.sleep_ms(50)

node = Node()
node.run()
```

---

# Part 1 — Send new data and listen (no forwarding)

### Goal

* Each node **injects** its own temperature **once per minute** (with jitter)
* Each node **listens** and prints received frames
* No rebroadcast yet

### Step 1A: Add `inject_own()` inside the class

Insert this under the “PART-SPECIFIC FUNCTIONS GO HERE” comment.

```python
    def inject_own(self):
        temp = read_temp_c()
        data = "{:.2f}".format(temp)

        msgid = str(time.ticks_ms() & 0xFFFFFFFF)
        frame = make_frame(NODE_ID, msgid, 0, "T", data)  # ttl not used yet
        self.advertise_frame(frame)
        print("INJECT:", frame)

        # schedule next inject: 60s + jitter
        self.next_inject_ms = time.ticks_add(
            time.ticks_ms(),
            (INJECT_PERIOD_S + self._rand_jitter_s()) * 1000
        )
```

### Step 1B: Add scan receive + print into `_irq`

Replace the `_irq` in the template with this:

```python
    def _irq(self, event, data):
        if event == _IRQ_SCAN_RESULT:
            addr_type, addr, adv_type, rssi, adv_data = data

            # Extract frame by searching for "M1|" inside adv payload
            try:
                raw = bytes(adv_data)
                idx = raw.find(b"M1|")
                if idx == -1:
                    return
                s = raw[idx:].decode("utf-8", "ignore").split("\x00")[0]
            except:
                return

            parsed = parse_frame(s)
            if not parsed:
                return

            orig, msgid, ttl, typ, payload = parsed
            print("RX (rssi={}): orig={} type={} data={}".format(rssi, orig, typ, payload))

        elif event == _IRQ_SCAN_DONE:
            self.scan()
```

### Step 1C: Add injection timer into `run()`

Replace `run()` in the template with:

```python
    def run(self):
        while True:
            now = time.ticks_ms()
            if time.ticks_diff(now, self.next_inject_ms) >= 0:
                self.inject_own()
            time.sleep_ms(50)
```

At this stage: students see messages, but they do **not** travel multi-hop.

---

# Part 2 — Forward received data (ignore duplicates and TTL)

### Goal

* Make multi-hop propagation obvious fast
* Intentionally “wrong”: no duplicate suppression, no TTL

### Step 2A: Add a `forward_raw()` function

Insert this under Part-specific functions:

```python
    def forward_raw(self, raw_frame_str):
        # Minimal forwarding (no TTL, no dedupe)
        # Add a tiny delay to reduce immediate collisions
        time.sleep_ms(10 + (time.ticks_ms() % 30))
        self.advertise_frame(raw_frame_str)
        print("FWD (raw):", raw_frame_str)
```

### Step 2B: In `_irq`, forward everything you receive

In the `_IRQ_SCAN_RESULT` handler (Part 1B), after printing RX, add:

```python
            # WARNING: this will cause a lot of repeats and loops
            self.forward_raw(s)
```

Now messages can travel multi-hop, but you will see storms/loops. That is expected in Part 2.

---

# Part 3 — Remove duplicates and include TTL

### Goal

* Add loop control mechanisms used in real mesh routing designs:

  * **De-duplication** (seen cache)
  * **TTL** (hop limit)

### Step 3A: Add dedupe helpers

Insert these under part-specific functions:

```python
    def seen_check_add(self, key):
        if key in self.seen:
            return True
        self.seen.append(key)
        if len(self.seen) > SEEN_MAX:
            del self.seen[0:len(self.seen) - SEEN_MAX]
        return False
```

### Step 3B: Change injection to include TTL

In `inject_own()`, change:

```python
        frame = make_frame(NODE_ID, msgid, 0, "T", data)
```

to:

```python
        frame = make_frame(NODE_ID, msgid, DEFAULT_TTL, "T", data)
        self.seen_check_add("{}:{}".format(NODE_ID, msgid))  # don't forward our own bounced packet
```

### Step 3C: Replace raw forwarding with TTL forwarding

Add this new forward function:

```python
    def forward_ttl(self, orig, msgid, ttl, typ, payload):
        ttl2 = ttl - 1
        if ttl2 < 0:
            return
        fwd = make_frame(orig, msgid, ttl2, typ, payload)

        time.sleep_ms(10 + (time.ticks_ms() % 30))
        self.advertise_frame(fwd)
        print("FWD ttl={}: {}".format(ttl2, fwd))
```

### Step 3D: Update `_irq` to dedupe + TTL forward

In `_IRQ_SCAN_RESULT`, after parsing:

1. Create dedupe key:

```python
            key = "{}:{}".format(orig, msgid)
            if self.seen_check_add(key):
                return
```

2. Print RX as “new”:

```python
            print("RX NEW (rssi={}): orig={} ttl={} type={} data={}".format(
                rssi, orig, ttl, typ, payload
            ))
```

3. Forward only if TTL allows:

```python
            if ttl > 0:
                self.forward_ttl(orig, msgid, ttl, typ, payload)
```

Now the behaviour is mesh-*like* flooding with basic loop control.

---

## What you should see in the console

* `INJECT:` once per minute per node (with jitter)
* `RX` / `RX NEW:` showing messages from other `orig=...`
* `FWD` showing that intermediate nodes are relaying data across the classroom
* When TTL is reduced, long-distance propagation becomes rarer

---

## Notes for a large class size

* Part 2 will get noisy quickly (intentionally).
* Part 3 is the “usable” version for the full class.
* Keep `DEFAULT_TTL` low (2–3) for stability.

---
