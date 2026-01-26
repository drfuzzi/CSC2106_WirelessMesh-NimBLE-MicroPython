## BLE Flooding “Mesh-like” Demo (MicroPython Pico W)

This lab uses **BLE advertisements** (broadcast) + **scanning** (receive). It is **not Bluetooth SIG BLE Mesh**; we are building a **mesh-like flooding relay** at the application layer.

### Key terms (use these consistently)

* **Inject**: originate a new message (new `MSGID`, `ORIG = me`)
* **Advertise burst**: transmit that message for a short time window (here **300 ms**) then stop
* **Forward**: advertise a message that originated from someone else

---

## Frame format

We encode frames as short text (kept small to fit in advertising name field):

```
M1|ORIG|MSGID|TTL|TYPE|DATA
```

Example:

```
M1|a1b2c3|12345678|3|T|26.73
```

---

# Starter template (copy first)

Create `mesh_node.py` with this template. It does **not** print RX yet and does **not** inject yet until you add Part 1 snippets.

```python
# mesh_node.py (STARTER TEMPLATE)
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
ADV_INTERVAL_US = 200_000      # advertising interval while active (200 ms)
ADV_BURST_MS    = 300          # advertise only 300 ms per injection/forward
SCAN_MS         = 10_000       # scan window (auto-restart)

INJECT_PERIOD_S = 60           # inject once every minute
INJECT_JITTER_S = 10           # add 0..10s jitter

DEFAULT_TTL     = 3            # used only in Part 3
SEEN_MAX        = 400          # used only in Part 3

NODE_ID = ubinascii.hexlify(machine.unique_id()).decode()[-6:]  # stable per-board ID

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
        return orig, msgid, int(ttl_s), typ, data
    except:
        return None

def adv_payload_name(name_str):
    name = name_str.encode()
    payload = bytearray(b"\x02\x01\x06")                 # Flags
    payload += bytearray((len(name) + 1, 0x09)) + name   # Complete Local Name
    return payload

def frame_to_name(frame):
    # Keep conservative so it fits reliably in advertising payload
    return frame[:25]

class Node:
    def __init__(self):
        self.ble = bluetooth.BLE()
        self.ble.active(True)
        self.ble.irq(self._irq)

        # ---------- Adv burst state ----------
        self._adv_active = False
        self._adv_stop_ms = 0

        # ---------- Part 3 dedupe ----------
        self.seen = []  # list of "orig:msgid"

        # ---------- Injection schedule ----------
        self.next_inject_ms = time.ticks_add(
            time.ticks_ms(),
            (INJECT_PERIOD_S + self._rand_jitter_s()) * 1000
        )

        self.scan()
        print("Node ID:", NODE_ID)

    def _rand_jitter_s(self):
        return urandom.getrandbits(8) % (INJECT_JITTER_S + 1)

    # ========== ADVERTISE: "burst" transmit then stop ==========
    def advertise_burst_start(self, frame, duration_ms=ADV_BURST_MS):
        payload = adv_payload_name(frame_to_name(frame))
        self.ble.gap_advertise(ADV_INTERVAL_US, adv_data=payload)
        self._adv_active = True
        self._adv_stop_ms = time.ticks_add(time.ticks_ms(), duration_ms)

    def advertise_burst_service(self):
        # Call frequently to stop advertising after the burst window ends
        if self._adv_active and time.ticks_diff(time.ticks_ms(), self._adv_stop_ms) >= 0:
            self.ble.gap_advertise(None)  # stop advertising
            self._adv_active = False

    def scan(self):
        self.ble.gap_scan(SCAN_MS, 30000, 30000)

    # ----------------------------
    # INSERT PART-SPECIFIC CODE BELOW
    # ----------------------------

    def _irq(self, event, data):
        # INSERT PART-SPECIFIC IRQ HANDLING HERE
        if event == _IRQ_SCAN_DONE:
            self.scan()

    def run(self):
        while True:
            # Always service advertising burst stop
            self.advertise_burst_service()

            # INSERT PART-SPECIFIC LOOP LOGIC HERE

            time.sleep_ms(20)

node = Node()
node.run()
```

What students see with just the template:

* Only: `Node ID: a1b2c3`
* Nothing else (by design)

---

# Part 1 — Send new data and listen (no forwarding)

Goal:

* Each node **injects** a new temperature frame **once per minute** (+ jitter)
* Each node **listens** and prints received frames
* **No forwarding** of received frames

### 1A) Add `inject_own()` inside the class

Insert under “INSERT PART-SPECIFIC CODE BELOW”.

```python
    def inject_own(self):
        temp = read_temp_c()
        data = "{:.2f}".format(temp)
        msgid = str(time.ticks_ms() & 0xFFFFFFFF)

        # TTL not used in Part 1; set 0 to make it clear there is no hop logic yet
        frame = make_frame(NODE_ID, msgid, 0, "T", data)

        # Advertise only once per injection = 300 ms burst
        self.advertise_burst_start(frame)
        print("INJECT:", frame)

        # Schedule next injection: 60s + jitter
        self.next_inject_ms = time.ticks_add(
            time.ticks_ms(),
            (INJECT_PERIOD_S + self._rand_jitter_s()) * 1000
        )
```

### 1B) Replace `_irq()` to print received frames

Replace the template `_irq()` with:

```python
    def _irq(self, event, data):
        if event == _IRQ_SCAN_RESULT:
            addr_type, addr, adv_type, rssi, adv_data = data
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

### 1C) Update `run()` to inject on schedule

Inside the `run()` loop, under “INSERT PART-SPECIFIC LOOP LOGIC HERE”, insert:

```python
            now = time.ticks_ms()
            if time.ticks_diff(now, self.next_inject_ms) >= 0:
                self.inject_own()
```

Expected behaviour:

* `INJECT:` appears roughly every 60–70 seconds (because of jitter)
* `RX:` appears when another node is injecting nearby

---

# Part 2 — Forward received data (ignore duplicates and TTL)

Goal:

* Make multi-hop propagation obvious
* Intentionally flawed: no dedupe, no TTL (expect “ping-pong” / storms)

### 2A) Add a simple forward function

Insert in the class:

```python
    def forward_raw(self, raw_frame_str):
        # Forward exactly what we received (intentionally wrong for Part 2)
        self.advertise_burst_start(raw_frame_str)
        print("FWD:", raw_frame_str)
```

### 2B) Modify `_irq()` to forward all received frames

In `_IRQ_SCAN_RESULT`, after printing RX, add:

```python
            self.forward_raw(s)
```

What students learn:

* One injected message can create repeated transmissions due to forwarding
* With 2 nodes, the packet can ping-pong indefinitely

---

# Part 3 — Remove duplicates and include TTL

Goal:

* Add two essential controls found in real mesh-like designs:

  1. **De-duplication** (`(ORIG, MSGID)` cache)
  2. **TTL** (hop limit, decremented on forward)

### 3A) Add dedupe helper

Insert in the class:

```python
    def seen_check_add(self, key):
        if key in self.seen:
            return True
        self.seen.append(key)
        if len(self.seen) > SEEN_MAX:
            del self.seen[0:len(self.seen) - SEEN_MAX]
        return False
```

### 3B) Update injection to include TTL and mark self as seen

Change Part 1 `inject_own()`:

* Replace TTL=0 with TTL=DEFAULT_TTL
* Add seen marking

```python
        frame = make_frame(NODE_ID, msgid, DEFAULT_TTL, "T", data)
        self.seen_check_add("{}:{}".format(NODE_ID, msgid))
        self.advertise_burst_start(frame)
        print("INJECT:", frame)
```

### 3C) Replace raw forwarding with TTL forwarding

Add:

```python
    def forward_ttl(self, orig, msgid, ttl, typ, payload):
        ttl2 = ttl - 1
        if ttl2 < 0:
            return
        fwd = make_frame(orig, msgid, ttl2, typ, payload)
        self.advertise_burst_start(fwd)
        print("FWD ttl={}: {}".format(ttl2, fwd))
```

### 3D) Update `_irq()` to dedupe + TTL forward

In `_IRQ_SCAN_RESULT`, after parsing:

1. Deduplicate:

```python
            key = "{}:{}".format(orig, msgid)
            if self.seen_check_add(key):
                return
```

2. Print as NEW:

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

What students learn:

* Dedupe stops infinite repeats
* TTL bounds propagation range (lower TTL → less reach)

---

## Notes for your 300 ms “advertise once per injection” requirement

* `advertise_burst_start(..., 300ms)` means: transmit for ~300 ms, then stop.
* It is not “one RF packet”, but it prevents the “same message advertised forever” effect.

If you want, I can provide a short “expected console output” section for each part (Part 1 / 2 / 3) so students can self-check quickly.
