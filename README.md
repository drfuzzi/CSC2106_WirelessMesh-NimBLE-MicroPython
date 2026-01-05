# MicroPython on Raspberry Pi Pico W **BLE Mesh (NimBLE)**

## Important Note on MicroPython BLE Mesh

The **MicroPython core build for the Pico W** currently exposes standard BLE **advertising and GATT services (Bluetooth Central/Peripheral)**, but it **does not include the full, native NimBLE Mesh networking stack** (which handles routing, provisioning, and self-healing) required for a true mesh network like the one in the original lab. Implementing a full mesh requires advanced custom C firmware development.

Therefore, this lab focuses on setting up a **Bi-Directional Peer-to-Peer BLE Communication** that acts as the building block for mesh networking.

## I. Objective and Prerequisites

### Objective

To set up two Pico W devices to communicate wirelessly using basic Bluetooth Low Energy (BLE) services, demonstrating the foundational data exchange necessary for any mesh-like IoT topology.

### Prerequisites

  * **Hardware**: Two Raspberry Pi Pico W boards.
  * **Software**: Thonny IDE.
  * **Environment**: Custom firmware or installation of necessary BLE libraries may be required for advanced mesh features beyond this basic peer-to-peer exchange.

## II. Peer-to-Peer BLE Setup (MicroPython)

We will configure one Pico W as a **BLE Peripheral (Advertiser)** to host data (Sensor Data) and the second Pico W as a **BLE Central (Scanner)** to connect and read that data.

### A. Pico W 1: The Peripheral (Sensor Node)

The Peripheral is like a Mesh Node advertising its sensor reading.

| File | Role | Code Summary |
| :--- | :--- | :--- |
| `peripheral.py` | Hosts the data service and advertises its presence. | Sets up a temperature characteristic (`UUID(0x2A6E)`), starts advertising, and updates the temperature value periodically. |

```python
# peripheral.py (Pico W 1 - Sender)
import bluetooth
import random
import time
import machine

# Define BLE UUIDs (using standard 16-bit UUIDs for temperature)
_IRQ_CENTRAL_CONNECT = const(1)
_IRQ_CENTRAL_DISCONNECT = const(2)
_TEMP_UUID = bluetooth.UUID(0x2A6E)
_ENV_SERVICE_UUID = bluetooth.UUID(0x181A) # Environmental Sensing Service

# Temperature characteristic setup: (UUID, Flags, Descriptors)
_TEMP_CHAR = (_TEMP_UUID, bluetooth.FLAG_READ | bluetooth.FLAG_NOTIFY)

# Service definition: (UUID, (Characteristic, ...))
_ENV_SERVICE = (_ENV_SERVICE_UUID, (_TEMP_CHAR,),)

class BLEPeripheral:
    def __init__(self, name):
        self._ble = bluetooth.BLE()
        self._ble.active(True)
        self._ble.irq(self._irq)
        ((self._handle,),) = self._ble.gatts_register_services((_ENV_SERVICE,))
        self._connections = set()
        self._advertise()
        print(f"Peripheral '{name}' started.")

    def _irq(self, event, data):
        if event == _IRQ_CENTRAL_CONNECT:
            conn_handle, _, _ = data
            self._connections.add(conn_handle)
            print("Central connected.")
        elif event == _IRQ_CENTRAL_DISCONNECT:
            conn_handle, _, _ = data
            self._connections.remove(conn_handle)
            self._advertise()
            print("Central disconnected. Advertising again.")

    def _advertise(self):
        # Advertising payload: Name and Service UUIDs
        self._ble.gap_advertise(100000, adv_data=bytearray(f"\x02\x01\x06\x03\x03\x1A\x18\x08\t{self._ble.config('mac')[0]}"))

    def update_temperature(self, temp_c):
        temp_bytes = str(temp_c).encode('utf-8')
        # Update the characteristic value
        self._ble.gatts_write(self._handle, temp_bytes)
        
        # Notify all connected Centrals
        for conn_handle in self._connections:
            self._ble.gatts_notify(conn_handle, self._handle)

# --- Main Logic ---
ble_peripheral = BLEPeripheral("PicoW_Sensor_A")
temp_sensor = machine.ADC(4)
conversion_factor = 3.3 / (65535)

def read_temperature():
    voltage = temp_sensor.read_u16() * conversion_factor
    temp = 27 - (voltage - 0.706) / 0.001721
    return temp

while True:
    current_temp = read_temperature()
    ble_peripheral.update_temperature(current_temp)
    print(f"Updating temperature: {current_temp:.2f} C. Connected clients: {len(ble_peripheral._connections)}")
    time.sleep(2)
```

### B. Pico W 2: The Central (Receiver Node)

The Central simulates a Mesh Node connecting to and reading data from its neighbor.

| File | Role | Code Summary |
| :--- | :--- | :--- |
| `central.py` | Scans for the Peripheral, connects to it, reads the characteristic value, and prints the data. | Searches for the Peripheral's advertised name, connects, discovers services, finds the temperature characteristic, and reads the value. |

```python
# central.py (Pico W 2 - Receiver/Scanner)
import bluetooth
import time
import ubinascii
from micropython import const

# Define BLE UUIDs (Must match Peripheral)
_IRQ_SCAN_RESULT = const(5)
_IRQ_SCAN_DONE = const(6)
_IRQ_GATTC_SERVICE_RESULT = const(9)
_IRQ_GATTC_SERVICE_DONE = const(10)
_IRQ_GATTC_CHARACTERISTIC_RESULT = const(11)
_IRQ_GATTC_CHARACTERISTIC_DONE = const(12)
_IRQ_GATTC_READ_DONE = const(15)
_IRQ_GATTC_NOTIFY = const(18)
_TEMP_UUID = bluetooth.UUID(0x2A6E)
_ENV_SERVICE_UUID = bluetooth.UUID(0x181A)

TARGET_NAME = "PicoW_Sensor_A"

class BLECentral:
    def __init__(self):
        self._ble = bluetooth.BLE()
        self._ble.active(True)
        self._ble.irq(self._irq)
        self._reset()

    def _reset(self):
        self._addr_type = None
        self._addr = None
        self._conn_handle = None
        self._temp_handle = None

    def _irq(self, event, data):
        if event == _IRQ_SCAN_RESULT:
            addr_type, addr, adv_type, rssi, adv_data = data
            if TARGET_NAME.encode() in adv_data:
                self._addr_type = addr_type
                self._addr = addr
                self._ble.gap_scan(None) # Stop scan
                print(f"Found target: {TARGET_NAME}")

        elif event == _IRQ_SCAN_DONE:
            if self._addr:
                print("Scan done. Connecting...")
                self._ble.gattc_connect(self._addr_type, self._addr)
            else:
                print("Scan done. Target not found.")
                self._ble.gap_scan(10000) # Scan again

        elif event == _IRQ_GATTC_SERVICE_RESULT:
            conn_handle, start_handle, end_handle, uuid = data
            if conn_handle == self._conn_handle and uuid == _ENV_SERVICE_UUID:
                self._start_handle = start_handle
                self._end_handle = end_handle
                print("Service found.")

        elif event == _IRQ_GATTC_SERVICE_DONE:
            if self._start_handle and self._end_handle:
                self._ble.gattc_discover_characteristics(
                    self._conn_handle, self._start_handle, self._end_handle
                )

        elif event == _IRQ_GATTC_CHARACTERISTIC_RESULT:
            conn_handle, start_handle, end_handle, uuid, properties = data
            if conn_handle == self._conn_handle and uuid == _TEMP_UUID:
                self._temp_handle = start_handle
                print("Temperature characteristic found.")

        elif event == _IRQ_GATTC_CHARACTERISTIC_DONE:
            if self._temp_handle:
                print("Discovery complete.")
            else:
                print("Characteristic not found.")
                self._ble.gattc_disconnect(self._conn_handle)

        elif event == _IRQ_GATTC_READ_DONE:
            conn_handle, value_handle, value = data
            if value_handle == self._temp_handle:
                temp = value.decode('utf-8')
                print(f"Received Temperature: {temp} C")

        elif event == _IRQ_GATTC_NOTIFY:
            conn_handle, value_handle, value = data
            if value_handle == self._temp_handle:
                temp = value.decode('utf-8')
                print(f"Notification Received: {temp} C")

        elif event == _IRQ_CENTRAL_CONNECT:
            conn_handle, addr_type, addr = data
            self._conn_handle = conn_handle
            self._ble.gattc_discover_services(self._conn_handle)
            print("Connected to Peripheral.")

        elif event == _IRQ_CENTRAL_DISCONNECT:
            print("Peripheral disconnected. Starting scan again.")
            self._reset()
            self.scan()


    def scan(self):
        self._ble.gap_scan(10000) # Scan for 10 seconds

    def read_temp(self):
        if self._conn_handle and self._temp_handle:
            self._ble.gattc_read(self._conn_handle, self._temp_handle)
            return True
        return False

# --- Main Logic ---
ble_central = BLECentral()
ble_central.scan()

while True:
    if ble_central.read_temp():
        time.sleep(5)
    else:
        time.sleep(1)
```

## III. Lab Assignment

### Basic BLE Peer-to-Peer Communication

  * **Task 1: Setup the Advertiser (Peripheral)**:
      * Upload and run `peripheral.py` on Pico W 1. Observe the terminal output to confirm it is advertising.
  * **Task 2: Setup the Scanner (Central)**:
      * Upload and run `central.py` on Pico W 2. Observe the terminal output. It should report finding, connecting to, and successfully receiving the temperature reading from Pico W 1.
  * **Task 3: Implement Actuator Control**:
      * Modify `peripheral.py` to toggle its LED (Pin **`"LED"`**) every time the temperature data is requested/read by the Central device. This confirms the connection and data exchange is active.
