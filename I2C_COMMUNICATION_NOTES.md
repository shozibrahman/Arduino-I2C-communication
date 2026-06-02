# I2C Communication Protocol — Complete Learning Notes
### Arduino I2C from Zero

---

## Table of Contents
1. [What is I2C?](#what-is-i2c)
2. [How I2C is Different From UART](#how-i2c-is-different-from-uart)
3. [The Two Wires](#the-two-wires)
4. [The Address System](#the-address-system)
5. [Master and Slave](#master-and-slave)
6. [Wiring](#wiring)
7. [I2C Buffer](#i2c-buffer)
8. [All Wire Functions](#all-wire-functions)
9. [Full Code — Master Requests, Slave Responds](#full-code--master-requests-slave-responds)
10. [Common Errors and Fixes](#common-errors-and-fixes)
11. [Key Takeaways](#key-takeaways)

---

## What is I2C?

**I2C = Inter Integrated Circuit**

A communication protocol that allows multiple devices to talk on just **2 wires**. One device is the **Master** (boss) and others are **Slaves** (workers).

Key word is **Synchronous** — unlike UART, I2C has a shared clock wire. Both sides are always in sync.

---

## How I2C is Different From UART

```
UART:
Arduino 1 TX ──────────────► Arduino 2 RX
Arduino 1 RX ◄────────────── Arduino 2 TX

Each Arduino talks independently.
No rules about who talks when.
No clock wire.

I2C:
Arduino 1 (MASTER) ◄──────► Arduino 2 (SLAVE)
          SDA ──────────────── SDA
          SCL ──────────────── SCL
          GND ──────────────── GND

MASTER is the boss.
SLAVE only speaks when MASTER asks.
Has a clock wire (SCL).
```

---

## The Two Wires

I2C only needs **2 wires** (plus GND):

```
SDA = Serial Data  → carries the actual data
SCL = Serial Clock → Master sends heartbeat pulses
                     both sides sync on each pulse
```

The Master provides the clock. Both sides read data on each pulse. No baud rate guessing needed.

```
SCL: ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌──
      └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘
       1   2   3   4   5   6   7   8   ← clock pulses

SDA: data changes on each pulse
```

---

## The Address System

This is what makes I2C special. Up to **127 devices** on just 2 wires — because every slave has a unique address.

```
Master ◄──── SDA/SCL ────► Slave (address 8)
                     ────► Slave (address 9)
                     ────► Slave (address 10)
                     ... up to 127 slaves!
```

When master wants to talk — it first sends the address:

```
Master says: "Address 8 — are you there?"
Slave 8:     "Yes! Here!"
Slave 9:     (stays silent)
Slave 10:    (stays silent)
```

---

## Master and Slave

### Real Life Analogy

```
MASTER = Teacher
SLAVE  = Student

Teacher: "Student number 8 — what is the altitude?"
Student 8: "18.50 meters"
Teacher: "Thank you."

Student NEVER speaks unless Teacher asks.
Student NEVER interrupts.
Teacher controls everything.
```

### Rules

```
Master:
  → starts every conversation
  → provides the clock (SCL)
  → decides who talks and when
  → calls Wire.begin() with no address

Slave:
  → never speaks first
  → only responds when master asks
  → has a fixed address
  → calls Wire.begin(address)
```

---

## Wiring

```
Arduino 1 (Master)          Arduino 2 (Slave)
──────────────────          ─────────────────
A4 (SDA) ◄──────────────►  A4 (SDA)
A5 (SCL) ──────────────►   A5 (SCL)
GND      ──────────────     GND

Pull-up resistors required:
5V ──── 4.7kΩ ──── A4 (SDA)
5V ──── 4.7kΩ ──── A5 (SCL)
```

> ⚠️ Pull-up resistors are required — without them I2C does not work reliably. They pull SDA and SCL HIGH when nobody is talking.

---

## I2C Buffer

### Size

```
UART buffer  →  64 bytes
I2C buffer   →  32 bytes  (half of UART)
```

### Big Difference From UART

```
UART:
  Bytes stay in buffer until YOU read them
  Safe to read anytime

I2C:
  Buffer gets WIPED automatically when next transmission starts
  Must read IMMEDIATELY inside onReceive() or onRequest()
```

### Safe Rule

```cpp
// ✅ CORRECT — read everything immediately
void receiveData(int numBytes) {
  Wire.readBytes((byte*)&altitude, 4);   // read right now!
}

// ❌ WRONG — trying to read later in loop()
void loop() {
  altitude = Wire.read();   // too late! buffer may be wiped
}
```

### Comparison Table

```
┌─────────────────┬──────────────┬──────────────────────┐
│                 │ UART         │ I2C                  │
├─────────────────┼──────────────┼──────────────────────┤
│ Buffer size     │ 64 bytes     │ 32 bytes             │
│ Buffer clears   │ When you     │ Automatically on     │
│                 │ read it      │ next transmission    │
│ When to read    │ Anytime      │ IMMEDIATELY in       │
│                 │              │ callback function    │
│ Miss a byte?    │ Still in     │ Gone forever ❌      │
│                 │ buffer ✅    │                      │
└─────────────────┴──────────────┴──────────────────────┘
```

---

## All Wire Functions

### Wire.begin()

Starts the I2C bus. How you call it determines Master or Slave:

```cpp
Wire.begin();      // → MASTER (no address)
Wire.begin(8);     // → SLAVE  (address = 8)
Wire.begin(50);    // → SLAVE  (address = 50)
```

---

### Wire.setClock()

Sets the speed of the SCL clock. Master only.

```cpp
Wire.setClock(100000);   // 100kHz — Standard mode (default)
Wire.setClock(400000);   // 400kHz — Fast mode (4x faster)
```

```
Standard mode: 100kHz → 1 byte takes ~90µs
Fast mode:     400kHz → 1 byte takes ~22µs

Always call AFTER Wire.begin()
Always call on MASTER only
```

---

### Wire.beginTransmission()

Master tells a specific slave to get ready to receive data:

```cpp
Wire.beginTransmission(8);    // talking to slave 8
```

```
Does NOT send data yet.
Just opens a conversation.
Think of it as dialing a phone number.
Always followed by Wire.write() then Wire.endTransmission()
```

---

### Wire.write()

Loads bytes into TX buffer. Must be called after beginTransmission():

```cpp
Wire.write(65);                          // send one byte
Wire.write('A');                         // send one character
Wire.write(data, 3);                     // send 3 bytes from array
Wire.write((byte*)&altitude, 4);         // send float as 4 bytes
```

```
Does not send immediately.
Loads into TX buffer (32 bytes max).
Actual sending happens at Wire.endTransmission()
```

---

### Wire.endTransmission()

Actually sends everything. Ends the conversation:

```cpp
Wire.endTransmission();        // send and release bus
Wire.endTransmission(false);   // send but KEEP bus open (use before requestFrom)
```

```
Returns status code:
  0 → success ✅
  2 → slave did not respond to address ❌
  3 → slave did not respond to data ❌
```

```cpp
// Good practice — check status
byte status = Wire.endTransmission();
if (status != 0) {
  Serial.print("Error: "); Serial.println(status);
}
```

---

### Wire.requestFrom()

Master asks slave to send data back:

```cpp
Wire.requestFrom(8, 4);      // ask slave 8 to send 4 bytes
```

```
BLOCKING — waits until slave responds
Takes ~110µs at 400kHz — safe for 4ms loop ✅

Returns number of bytes actually received:
  4 → slave responded ✅
  0 → slave did not respond ❌
```

---

### Wire.available()

Checks how many bytes are waiting in RX buffer:

```cpp
int count = Wire.available();
// count = 4  → 4 bytes waiting
// count = 0  → nothing waiting
```

---

### Wire.read()

Reads ONE byte from RX buffer and removes it:

```cpp
byte b = Wire.read();
```

```
Buffer before:  [10][20][30][40]
Wire.read()  →  b = 10
Buffer after:   [20][30][40]
```

---

### Wire.readBytes()

Reads multiple bytes at once:

```cpp
byte buf[4];
Wire.readBytes(buf, 4);                  // read into array
Wire.readBytes((byte*)&altitude, 4);     // read directly into float
```

---

### Wire.onReceive()

Slave only. Function that fires when master sends data:

```cpp
Wire.onReceive(receiveData);   // register in setup()

void receiveData(int numBytes) {
  // read bytes immediately here!
  byte b = Wire.read();
}
```

> ⚠️ Read ALL bytes immediately. No delay() inside.

---

### Wire.onRequest()

Slave only. Function that fires when master calls requestFrom():

```cpp
Wire.onRequest(sendData);   // register in setup()

void sendData() {
  Wire.write((byte*)&altitude, 4);   // send immediately!
}
```

> ⚠️ Send bytes immediately. No delay() inside.

---

### All Functions Summary

```
┌─────────────────────────┬──────────┬──────────┬────────────────────────────┐
│ Function                │ Master   │ Slave    │ What it does               │
├─────────────────────────┼──────────┼──────────┼────────────────────────────┤
│ Wire.begin()            │ ✅       │ ✅       │ Start I2C as master        │
│ Wire.begin(addr)        │          │ ✅       │ Start I2C as slave         │
│ Wire.setClock(speed)    │ ✅       │          │ Set bus speed              │
│ Wire.beginTransmission()│ ✅       │          │ Open talk to slave         │
│ Wire.write()            │ ✅       │ ✅       │ Load bytes to send         │
│ Wire.endTransmission()  │ ✅       │          │ Actually send bytes        │
│ Wire.requestFrom()      │ ✅       │          │ Ask slave for bytes        │
│ Wire.available()        │ ✅       │ ✅       │ How many bytes waiting     │
│ Wire.read()             │ ✅       │ ✅       │ Read one byte from buffer  │
│ Wire.readBytes()        │ ✅       │ ✅       │ Read many bytes at once    │
│ Wire.onReceive()        │          │ ✅       │ Register receive callback  │
│ Wire.onRequest()        │          │ ✅       │ Register request callback  │
└─────────────────────────┴──────────┴──────────┴────────────────────────────┘
```

---

## Full Code — Master Requests, Slave Responds

### How it Works

```
Master                          Slave (address 8)
──────                          ─────────────────

Wire.requestFrom(8, 4)
→ "Hey slave 8, give me 4 bytes"
                                → Wire.onRequest() fires!
                                → Wire.write(altitude bytes)

Wire.readBytes(&altitude, 4)
→ reads the 4 bytes ✅
```

### Arduino 2 — Slave

```cpp
#include <Wire.h>

float altitude = 18.50;

void setup() {
  Wire.begin(8);                  // slave address = 8
  Wire.onRequest(sendAltitude);   // register callback
  Serial.begin(115200);
}

void sendAltitude() {
  Wire.write((byte*)&altitude, 4);   // send 4 bytes of float
}

void loop() {
  // update altitude from real sensor here
  // altitude = bmp.readAltitude();
  delay(10);
}
```

### Arduino 1 — Master (4ms loop)

```cpp
#include <Wire.h>

float altitude = 0.0;
bool  newData  = false;

void setup() {
  Wire.begin();
  Wire.setClock(400000);
  Serial.begin(115200);
}

void loop() {
  unsigned long loopStart = micros();

  // Request 4 bytes from slave 8
  Wire.requestFrom(8, 4);

  if (Wire.available() == 4) {
    Wire.readBytes((byte*)&altitude, 4);
    newData = true;
  }

  if (newData) {
    newData = false;
    Serial.print("Altitude: ");
    Serial.println(altitude);
  }

  // Other tasks here
  // runMotors();
  // updatePID();

  while (micros() - loopStart < 4000);
}
```

### Timing

```
At 400kHz fast mode:
4 bytes × 9 bits = 36 bits
36 bits / 400,000 = 90µs + overhead ≈ 110µs total

4ms loop budget:
├── 0.11ms → full I2C request + receive
└── 3.89ms → FREE for other tasks ✅
```

---

## Understanding (byte*)&altitude

This is used everywhere in I2C float sending:

```cpp
Wire.write((byte*)&altitude, 4);
```

Break it into 3 parts:

```
&altitude   → address of the variable in memory
(byte*)     → treat that address as raw bytes
, 4         → read 4 bytes from that address

Wire.write cannot send a float directly.
So you open it up and send its 4 raw bytes.
Receiver collects the 4 bytes and rebuilds the float.
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| Nothing received | Missing pull-up resistors | Add 4.7kΩ on SDA and SCL |
| endTransmission returns 2 | Wrong slave address | Check address matches Wire.begin(addr) |
| Corrupted data | Reading outside onReceive | Read bytes immediately inside callback |
| Slave not responding | Not calling Wire.begin(addr) | Make sure slave has Wire.begin with address |
| requestFrom returns 0 | Slave has no onRequest | Register Wire.onRequest(func) on slave |

---

## Key Takeaways

```
1. I2C uses only 2 wires + GND
   SDA = data, SCL = clock

2. Master controls everything
   Slave only responds when asked

3. Every slave has a unique address (1–127)
   Master addresses slave before talking

4. I2C buffer = 32 bytes
   Read immediately inside callbacks
   Buffer wipes on next transmission

5. Wire.write() loads data — does not send yet
   Wire.endTransmission() actually sends

6. Wire.endTransmission(false) keeps bus open
   Use before Wire.requestFrom()

7. Pull-up resistors 4.7kΩ are required
   On both SDA and SCL lines

8. Wire.requestFrom() is blocking ~110µs
   Still safe for 4ms loop
```

---

