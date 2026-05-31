# Dispatcher Loop Architecture

## Overview

`Dispatcher::loop()` is the **main event loop** for all MeshCore nodes (repeaters, companion radios, room servers, sensors). It manages radio I/O, packet queuing, transmission scheduling, and channel access coordination.

## Architecture Position

```
┌─────────────────────────────────────────────────────────────────┐
│  Application Layer                                              │
│  (MyMesh, simple_repeater, simple_sensor, etc.)                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  Protocol Layer: Mesh / BaseChatMesh                            │
│  (Routing, encryption, peer management, message handling)       │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  Dispatch Layer: Dispatcher ◄── YOU ARE HERE                    │
│  (Radio I/O, packet scheduling, airtime budget, LBT)            │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  Radio Driver: Radio (RadioLib wrapper)                         │
│  (SX1262, SX1276, etc. hardware abstraction)                    │
└─────────────────────────────────────────────────────────────────┘
```

## Main Loop Flow

`Dispatcher::loop()` in [src/Dispatcher.cpp:42-109](../src/Dispatcher.cpp#L42-L109)

```cpp
void Dispatcher::loop() {
  // 1. Noise floor calibration (every 2 seconds)
  // 2. Radio driver loop()
  // 3. Check if radio stuck in non-RX mode > 8 seconds
  // 4. Handle outbound packet completion
  // 5. Process delayed inbound packets (score-based delay)
  // 6. Check for new received packets → checkRecv()
  // 7. Check for packets to send → checkSend()
}
```

### Step-by-Step Execution

#### 1. Noise Floor Calibration

```cpp
if (millisHasNowPassed(next_floor_calib_time)) {
  _radio->triggerNoiseFloorCalibrate(getInterferenceThreshold());
  next_floor_calib_time = futureMillis(NOISE_FLOOR_CALIB_INTERVAL);
}
```

- **Purpose**: Periodically recalibrate radio's noise floor for accurate RSSI/SNR measurements
- **Interval**: 2 seconds (default)
- **Why**: Radio environment changes over time; calibration maintains signal quality accuracy

#### 2. Radio Driver Loop

```cpp
_radio->loop();
```

- **Purpose**: Give radio driver time to handle internal state machine (SPI transactions, interrupts, etc.)
- **Driver examples**: RadioLib-based implementations for SX1262, SX1276, RFM95, etc.

#### 3. Radio State Monitoring

```cpp
bool is_recv = _radio->isInRecvMode();
if (is_recv != prev_isrecv_mode) {
  prev_isrecv_mode = is_recv;
  if (!is_recv) {
    radio_nonrx_start = _ms->getMillis();
  }
}
if (!is_recv && _ms->getMillis() - radio_nonrx_start > 8000) {
  _err_flags |= ERR_EVENT_STARTRX_TIMEOUT;
}
```

- **Purpose**: Detect if radio gets "stuck" in transmit or idle mode
- **Threshold**: 8 seconds without being in RX mode
- **Action**: Set error flag `ERR_EVENT_STARTRX_TIMEOUT`
- **Why**: Indicates radio driver malfunction or hardware issue

#### 4. Outbound Packet Completion

```cpp
if (outbound) {
  if (_radio->isSendComplete()) {
    long t = _ms->getMillis() - outbound_start;
    total_air_time += t;
    next_tx_time = futureMillis(t * getAirtimeBudgetFactor());  // 2.0 = 33% duty cycle

    _radio->onSendFinished();
    logTx(outbound, 2 + outbound->path_len + outbound->payload_len);
    releasePacket(outbound);
    outbound = NULL;
  } else if (millisHasNowPassed(outbound_expiry)) {
    // Transmission timeout (1.5× estimated airtime)
    _radio->onSendFinished();
    logTxFail(outbound, 2 + outbound->path_len + outbound->payload_len);
    releasePacket(outbound);
    outbound = NULL;
  } else {
    return;  // Still transmitting, can't do other radio activity
  }

  next_agc_reset_time = futureMillis(getAGCResetInterval());
}
```

- **Purpose**: Handle completion of ongoing transmission
- **Success path**:
  1. Record actual airtime
  2. Calculate next TX time based on airtime budget
  3. Reset radio AGC (Automatic Gain Control)
  4. Release packet back to pool
- **Failure path**: Timeout after 1.5× estimated airtime
- **Blocking**: While `outbound != NULL`, no other radio activity can occur

#### 5. Delayed Inbound Queue Processing

```cpp
Packet* pkt = _mgr->getNextInbound(_ms->getMillis());
if (pkt) {
  processRecvPacket(pkt);
}
```

- **Purpose**: Process FLOOD packets after their score-based delay expires
- **Why delayed**: Spread retransmissions over time to reduce collisions
- **Queue**: Priority queue ordered by due time

#### 6. Check for Received Packets

```cpp
checkRecv();
```

See [Receive Path](#receive-path-checkrecv) below.

#### 7. Check for Packets to Send

```cpp
checkSend();
```

See [Send Path](#send-path-checksend) below.

---

## Receive Path: `checkRecv()`

[src/Dispatcher.cpp:111-210](../src/Dispatcher.cpp#L111-L210)

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Poll radio for incoming data                                 │
│    _radio->recvRaw(raw, MAX_TRANS_UNIT)                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Parse raw bytes into Packet structure                        │
│    - header, transport_codes, path_len, path[], payload[]       │
│    - Validate lengths                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Capture signal quality metrics                               │
│    - SNR (signal-to-noise ratio)                                │
│    - RSSI (received signal strength)                            │
│    - Quality score: _radio->packetScore(snr, len)               │
│    - Estimated airtime: _radio->getEstAirtimeFor(len)           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Routing decision based on packet type                        │
│                                                                 │
│    ┌──────────────────┐          ┌─────────────────┐           │
│    │  FLOOD packet?   │──Yes───▶ │ Score-based     │           │
│    │  (broadcast)     │          │ delay queue     │           │
│    └────────┬─────────┘          │                 │           │
│             │                    │ Good signal     │           │
│             No                   │ (high score)    │           │
│             │                    │ → 50-200ms      │           │
│             ▼                    │                 │           │
│    ┌──────────────────┐          │ Bad signal      │           │
│    │  DIRECT packet   │          │ (low score)     │           │
│    │  (known path)    │          │ → up to 32s     │           │
│    └────────┬─────────┘          └─────────────────┘           │
│             │                                                   │
│             ▼                                                   │
│    ┌──────────────────┐                                        │
│    │ Process          │                                        │
│    │ immediately      │                                        │
│    └──────────────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Code Structure

```cpp
void Dispatcher::checkRecv() {
  Packet* pkt;
  float score;
  uint32_t air_time;

  // 1. Poll radio
  {
    uint8_t raw[MAX_TRANS_UNIT+1];
    int len = _radio->recvRaw(raw, MAX_TRANS_UNIT);
    if (len > 0) {
      logRxRaw(_radio->getLastSNR(), _radio->getLastRSSI(), raw, len);

      // 2. Allocate packet from pool
      pkt = _mgr->allocNew();
      if (pkt == NULL) {
        // Pool exhausted, drop packet
        return;
      }

      // 3. Parse raw bytes
      int i = 0;
      pkt->header = raw[i++];
      if (pkt->hasTransportCodes()) {
        memcpy(&pkt->transport_codes[0], &raw[i], 2); i += 2;
        memcpy(&pkt->transport_codes[1], &raw[i], 2); i += 2;
      }
      pkt->path_len = raw[i++];

      // 4. Validate
      if (pkt->path_len > MAX_PATH_SIZE || i + pkt->path_len > len) {
        _mgr->free(pkt);  // Invalid, drop packet
        return;
      }

      // 5. Copy path and payload
      memcpy(pkt->path, &raw[i], pkt->path_len); i += pkt->path_len;
      pkt->payload_len = len - i;
      memcpy(pkt->payload, &raw[i], pkt->payload_len);

      // 6. Signal quality
      pkt->_snr = _radio->getLastSNR() * 4.0f;
      score = _radio->packetScore(_radio->getLastSNR(), len);
      air_time = _radio->getEstAirtimeFor(len);
      rx_air_time += air_time;
    }
  }

  // 7. Process or delay
  if (pkt) {
    logRx(pkt, pkt->getRawLength(), score);

    if (pkt->isRouteFlood()) {
      n_recv_flood++;

      int _delay = calcRxDelay(score, air_time);
      if (_delay < 50) {
        processRecvPacket(pkt);  // Good signal, process immediately
      } else {
        if (_delay > MAX_RX_DELAY_MILLIS) {
          _delay = MAX_RX_DELAY_MILLIS;  // Cap at 32 seconds
        }
        _mgr->queueInbound(pkt, futureMillis(_delay));  // Queue for later
      }
    } else {
      n_recv_direct++;
      processRecvPacket(pkt);  // DIRECT packets always immediate
    }
  }
}
```

### Score-Based Delay Algorithm

The delay calculation prevents "thundering herd" retransmissions:

```cpp
int Dispatcher::calcRxDelay(float score, uint32_t air_time) const {
  return (int) ((pow(10, 0.85f - score) - 1.0) * air_time);
}
```

**Score ranges** (0.0 to 1.0):
- **Score 0.85+**: Delay < 50ms → process immediately
- **Score 0.7**: Delay ≈ 200ms × airtime
- **Score 0.5**: Delay ≈ 2 seconds × airtime
- **Score 0.3**: Delay ≈ 30 seconds × airtime (capped at 32s)

**Example**: For a 1-second airtime packet:
- Excellent SNR (+10dB, score=0.9): 0ms delay
- Good SNR (+5dB, score=0.75): 178ms delay
- Fair SNR (0dB, score=0.5): 3.2s delay
- Poor SNR (-10dB, score=0.3): 32s delay (capped)

**Purpose**: Nodes with better signal quality retransmit first. Distant nodes with poor reception wait longer, allowing them to hear the better-quality retransmission and cancel their own.

### Packet Processing

```cpp
void Dispatcher::processRecvPacket(Packet* pkt) {
  DispatcherAction action = onRecvPacket(pkt);  // Virtual method, overridden by Mesh

  if (action == ACTION_RELEASE) {
    _mgr->free(pkt);  // Done with packet
  } else if (action == ACTION_MANUAL_HOLD) {
    // Subclass holds packet, will call releasePacket() later
  } else {
    // ACTION_RETRANSMIT* - queue for retransmission
    uint8_t priority = (action >> 24) - 1;
    uint32_t _delay = action & 0xFFFFFF;
    _mgr->queueOutbound(pkt, priority, futureMillis(_delay));
  }
}
```

The `onRecvPacket()` virtual method is where `Mesh` class implements routing logic:
- Deduplication check
- Decryption
- Path building/updating
- Routing decision (forward, deliver locally, drop)

---

## Send Path: `checkSend()`

[src/Dispatcher.cpp:226-297](../src/Dispatcher.cpp#L226-L297)

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Pre-flight checks                                            │
│    ✓ Packets waiting in queue?                                  │
│    ✓ Airtime budget satisfied (next_tx_time passed)?            │
│    ✓ Channel clear? (Listen Before Talk - LBT)                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Get next packet from priority queue                          │
│    outbound = _mgr->getNextOutbound(_ms->getMillis())           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Serialize packet into raw bytes                              │
│    header + transport_codes + path_len + path[] + payload[]     │
│    Validate: total_len ≤ MAX_TRANS_UNIT                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Start radio transmission (non-blocking)                      │
│    _radio->startSendRaw(raw, len)                               │
│    Set outbound_expiry = now + (1.5 × estimated_airtime)        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Return to main loop                                          │
│    (Completion handled in loop() step 4)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Code Structure

```cpp
void Dispatcher::checkSend() {
  // 1. Pre-flight checks
  if (_mgr->getOutboundCount(_ms->getMillis()) == 0) return;  // Nothing to send
  if (!millisHasNowPassed(next_tx_time)) return;  // Still in radio silence period

  // 2. Listen Before Talk (LBT) - Channel Activity Detection
  if (_radio->isReceiving()) {
    if (cad_busy_start == 0) {
      cad_busy_start = _ms->getMillis();  // Start tracking busy duration
    }

    if (_ms->getMillis() - cad_busy_start > getCADFailMaxDuration()) {  // 4 seconds
      _err_flags |= ERR_EVENT_CAD_TIMEOUT;
      // Force transmission anyway (radio might be stuck)
    } else {
      next_tx_time = futureMillis(getCADFailRetryDelay());  // 200ms retry
      return;  // Channel busy, wait and retry
    }
  }
  cad_busy_start = 0;  // Channel clear

  // 3. Get next packet
  outbound = _mgr->getNextOutbound(_ms->getMillis());
  if (outbound) {
    // 4. Serialize into raw bytes
    int len = 0;
    uint8_t raw[MAX_TRANS_UNIT];

    raw[len++] = outbound->header;
    if (outbound->hasTransportCodes()) {
      memcpy(&raw[len], &outbound->transport_codes[0], 2); len += 2;
      memcpy(&raw[len], &outbound->transport_codes[1], 2); len += 2;
    }
    raw[len++] = outbound->path_len;
    memcpy(&raw[len], outbound->path, outbound->path_len); len += outbound->path_len;

    // 5. Validate length
    if (len + outbound->payload_len > MAX_TRANS_UNIT) {
      _mgr->free(outbound);  // Invalid, drop packet
      outbound = NULL;
      return;
    }

    memcpy(&raw[len], outbound->payload, outbound->payload_len);
    len += outbound->payload_len;

    // 6. Start transmission
    uint32_t max_airtime = _radio->getEstAirtimeFor(len) * 3 / 2;  // 1.5× estimate
    outbound_start = _ms->getMillis();
    bool success = _radio->startSendRaw(raw, len);

    if (!success) {
      logTxFail(outbound, outbound->getRawLength());
      releasePacket(outbound);
      outbound = NULL;
      return;
    }

    outbound_expiry = futureMillis(max_airtime);  // Timeout for tx completion
  }
}
```

### Listen Before Talk (LBT)

```cpp
if (_radio->isReceiving()) {
  // Channel is busy - someone else is transmitting or receiving
  if (cad_busy_start == 0) {
    cad_busy_start = _ms->getMillis();
  }

  if (_ms->getMillis() - cad_busy_start > getCADFailMaxDuration()) {  // 4s default
    _err_flags |= ERR_EVENT_CAD_TIMEOUT;
    // Force transmission anyway
  } else {
    next_tx_time = futureMillis(getCADFailRetryDelay());  // 200ms default
    return;  // Wait and retry
  }
}
```

**Purpose**: Prevent collisions by waiting for channel to be clear before transmitting.

**CAD (Channel Activity Detection)**:
- Check if radio is mid-receive or detects preamble
- If busy, retry after 200ms (default)
- Timeout after 4 seconds (prevents deadlock if radio stuck)

**Note**: Some LoRa configurations don't support CAD; in those cases `isReceiving()` always returns false and packets transmit immediately.

---

## Airtime Budget Management

### Purpose

Limit channel utilization to:
1. **Comply with regulations**: Many regions limit duty cycle to 1-10% on unlicensed ISM bands
2. **Fair channel access**: Prevent single node from monopolizing the channel
3. **Reduce collisions**: More idle time = more opportunities for LBT to detect traffic

### Formula

```cpp
void Dispatcher::loop() {
  if (outbound && _radio->isSendComplete()) {
    long actual_airtime = _ms->getMillis() - outbound_start;
    total_air_time += actual_airtime;

    float budget_factor = getAirtimeBudgetFactor();  // Default: 2.0
    next_tx_time = futureMillis(actual_airtime * budget_factor);

    // ... cleanup ...
  }
}
```

**Default configuration**:
- `getAirtimeBudgetFactor() = 2.0`
- **Duty cycle**: 1 / (1 + factor) = 1 / (1 + 2) = **33.3%**

**Example**:
- Packet takes 1 second to transmit
- Radio must remain silent for 2 seconds (2.0 × 1s)
- Next transmission earliest at T+3 seconds
- Duty cycle: 1s active / 3s total = 33.3%

### Customization

Override `getAirtimeBudgetFactor()` in your subclass:

```cpp
float MyDispatcher::getAirtimeBudgetFactor() const {
  return 9.0;  // 1/(1+9) = 10% duty cycle (more conservative)
}
```

---

## Packet Queue Management

### Outbound Queue

**Priority levels**: 0 (highest) to 255 (lowest)

```cpp
void Dispatcher::sendPacket(Packet* packet, uint8_t priority, uint32_t delay_millis) {
  _mgr->queueOutbound(packet, priority, futureMillis(delay_millis));
}
```

**Typical priorities**:
- **0**: Urgent ACKs
- **1**: Direct messages (TXT_MSG, REQ, RESPONSE)
- **2**: Group messages (GRP_TXT)
- **3**: Path advertisements
- **4**: ADVERTs, TRACEs

**Selection**: `getNextOutbound()` returns highest priority packet whose due time has passed.

### Inbound Queue (Delayed)

Used for FLOOD packets with score-based delay:

```cpp
_mgr->queueInbound(pkt, futureMillis(delay_millis));
```

**Queue order**: By due time (not priority)

**Processing**: Main loop checks `getNextInbound()` every iteration, processes any packets whose due time has passed.

### Packet Pool

Fixed-size pool to avoid dynamic allocation:

```cpp
Packet* Dispatcher::obtainNewPacket() {
  auto pkt = _mgr->allocNew();
  if (pkt == NULL) {
    _err_flags |= ERR_EVENT_FULL;  // Pool exhausted
  }
  return pkt;
}

void Dispatcher::releasePacket(Packet* packet) {
  _mgr->free(packet);
}
```

**Pool size**: Configured per platform (typically 16-32 packets)

**Exhaustion handling**: Drop incoming packets when pool full

---

## Error Monitoring

### Error Flags

```cpp
// src/Dispatcher.h
#define ERR_EVENT_STARTRX_TIMEOUT  0x01  // Radio stuck in non-RX mode
#define ERR_EVENT_CAD_TIMEOUT      0x02  // Channel busy too long
#define ERR_EVENT_FULL             0x04  // Packet pool exhausted
```

### Checking Errors

```cpp
uint8_t errors = dispatcher.getErrorFlags();
if (errors & ERR_EVENT_STARTRX_TIMEOUT) {
  // Radio stuck - may need reset
}
if (errors & ERR_EVENT_CAD_TIMEOUT) {
  // Channel very congested or radio stuck
}
if (errors & ERR_EVENT_FULL) {
  // Too much traffic - consider increasing packet pool size
}

dispatcher.clearErrorFlags();
```

---

## Virtual Methods (Override Points)

### `onRecvPacket()`

```cpp
virtual DispatcherAction onRecvPacket(Packet* pkt) = 0;
```

**Purpose**: Handle received packet (routing, decryption, delivery)

**Return values**:
- `ACTION_RELEASE`: Done with packet, release to pool
- `ACTION_MANUAL_HOLD`: Keep packet, will call `releasePacket()` later
- `ACTION_RETRANSMIT(priority, delay)`: Queue for retransmission

**Implemented by**: `Mesh` class in [src/Mesh.cpp](../src/Mesh.cpp)

### Configuration Methods

```cpp
virtual float getAirtimeBudgetFactor() const;       // Default: 2.0 (33% duty cycle)
virtual int calcRxDelay(float score, uint32_t air_time) const;  // Score-based delay
virtual uint32_t getCADFailRetryDelay() const;      // Default: 200ms
virtual uint32_t getCADFailMaxDuration() const;     // Default: 4000ms (4s)
virtual uint32_t getAGCResetInterval() const;       // Default: 0 (disabled)
virtual float getInterferenceThreshold() const;     // For noise floor calib
```

### Logging Hooks

```cpp
virtual void logRxRaw(float snr, float rssi, const uint8_t* raw, int len);
virtual void logRx(Packet* pkt, int raw_len, float score);
virtual void logTx(Packet* pkt, int raw_len);
virtual void logTxFail(Packet* pkt, int raw_len);
```

**Purpose**: Optional logging for debugging and analysis

**Default implementation**: No-op (override to add logging)

---

## Timing Utilities

### Millisecond Overflow Handling

```cpp
bool Dispatcher::millisHasNowPassed(unsigned long timestamp) const {
  return (long)(_ms->getMillis() - timestamp) > 0;
}
```

**Purpose**: Correctly handles when `millis()` wraps around to zero (~49 days uptime)

**How it works**: 2's complement arithmetic makes unsigned subtraction work for differences up to half the word size (2^31 milliseconds ≈ 24 days)

### Future Time Calculation

```cpp
unsigned long Dispatcher::futureMillis(int millis_from_now) const {
  return _ms->getMillis() + millis_from_now;
}
```

**Purpose**: Calculate future timestamp, correctly handles overflow

---

## Code References

### Core Files

- **Main loop**: [src/Dispatcher.cpp:42-109](../src/Dispatcher.cpp#L42-L109)
- **Receive path**: [src/Dispatcher.cpp:111-210](../src/Dispatcher.cpp#L111-L210)
- **Send path**: [src/Dispatcher.cpp:226-297](../src/Dispatcher.cpp#L226-L297)
- **Packet processing**: [src/Dispatcher.cpp:212-224](../src/Dispatcher.cpp#L212-L224)
- **Public API**: [src/Dispatcher.cpp:299-321](../src/Dispatcher.cpp#L299-L321)
- **Header**: [src/Dispatcher.h](../src/Dispatcher.h)

### Mesh Implementation

- **Routing logic**: [src/Mesh.cpp](../src/Mesh.cpp)
- **Packet structure**: [src/Packet.h](../src/Packet.h)

### Radio Drivers

- **RadioLib wrappers**: [src/helpers/radiolib/](../src/helpers/radiolib/)

---

## Example: Packet Lifecycle

### Sending a Message

```
Application (MyMesh)
  │
  ├─ createTextMessage()
  │   └─ Encrypt with AES-128 + HMAC
  │
  ├─ obtainNewPacket()  ──────────────┐
  │                                   │ Get packet from pool
  ├─ Fill packet structure            │
  │   - header, path, payload         │
  │                                   │
  ├─ sendPacket(pkt, priority=1, delay=0)
  │   └─ _mgr->queueOutbound()  ──────┤ Add to outbound queue
  │                                   │
  ▼                                   │
Dispatcher::loop()                    │
  │                                   │
  ├─ checkSend()  ─────────────────┐  │
  │   ├─ Check airtime budget      │  │
  │   ├─ Check LBT (channel clear) │  │
  │   ├─ getNextOutbound() ◄───────┼──┘ Get from queue
  │   ├─ Serialize to raw bytes    │
  │   └─ _radio->startSendRaw() ───┼────▶ Radio (non-blocking)
  │                                │
  │ (next iteration)               │
  ├─ _radio->isSendComplete() ◄───┼──── Radio signals completion
  │   ├─ Record airtime            │
  │   ├─ Calculate next_tx_time    │
  │   └─ releasePacket() ──────────┼────▶ Return to pool
  │                                │
  └────────────────────────────────┘
```

### Receiving a Message

```
Radio hardware receives LoRa packet
  │
  ▼
Dispatcher::loop()
  │
  ├─ checkRecv()
  │   ├─ _radio->recvRaw() ◄──────────── Poll radio
  │   ├─ allocNew() ───────────────┐
  │   │                            │ Get packet from pool
  │   ├─ Parse raw bytes           │
  │   ├─ Capture SNR, RSSI, score  │
  │   │                            │
  │   └─ FLOOD packet? ────┬───────┘
  │                        │
  │         Yes            │ No (DIRECT)
  │         │              │
  │         ▼              ▼
  │   queueInbound()  processRecvPacket()
  │   (with delay)        │
  │         │             │
  │         │             ▼
  │         │        Mesh::onRecvPacket()
  │         │             ├─ Deduplication check
  │         │             ├─ Decrypt with AES-128
  │         │             ├─ Routing decision
  │         │             └─ Return ACTION_*
  │         │                  │
  │         │                  ├─ ACTION_RELEASE ───▶ releasePacket()
  │         │                  │                      (return to pool)
  │         │                  │
  │         │                  └─ ACTION_RETRANSMIT
  │         │                      └─ queueOutbound()
  │         │
  │ (later iteration)
  │
  ├─ getNextInbound() ◄───┘
  │   └─ processRecvPacket()
  │       └─ (same as above)
  │
  └──────────────────────────────
```

---

## Performance Characteristics

### Timing

- **Main loop frequency**: Typically 100-1000 Hz (1-10ms per iteration)
- **Radio poll latency**: 10-50ms (time between packet arrival and `recvRaw()` detecting it)
- **Transmission delay**: Airtime budget + LBT retry + queue delay
  - Typical: 100-500ms for short packets
  - Congested: 1-5 seconds

### Memory

- **Packet pool**: 16-32 packets × ~300 bytes = 5-10 KB
- **Queue overhead**: Minimal (indices only)
- **Stack usage**: <1 KB per loop iteration

### CPU

- **Idle time**: >95% (waiting for radio)
- **Crypto**: ~5ms per packet (AES-128 + HMAC)
- **Serialization**: <1ms per packet

---

## Comparison: Dispatcher vs Mesh

| Layer | Responsibility | Abstraction Level |
|-------|----------------|-------------------|
| **Dispatcher** | Radio I/O, scheduling, airtime, LBT | Low-level (bytes, timing) |
| **Mesh** | Routing, encryption, peer management | Protocol-level (messages, paths) |

**Dispatcher doesn't know about**:
- Message types (TXT_MSG, ADVERT, etc.)
- Encryption/decryption
- Peer identities
- Routing algorithms
- Deduplication

**Dispatcher only knows about**:
- Raw bytes in/out
- Packet header format
- Timing and scheduling
- Signal quality
- Channel access

This separation allows Dispatcher to be reused for different protocols (not just MeshCore mesh networking).
