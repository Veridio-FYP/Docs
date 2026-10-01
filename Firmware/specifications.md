# ESP32-CAM Kiosk: Low-Level Technical Specification

## 1. Role of the kiosk

One kiosk is fixed in each room. It is the trust anchor between the physical world and the backend. It does five jobs:

1. Decode the student's rotating QR code at 10–12 cm.
2. Prove presence by broadcasting a BLE beacon the app uses for 5 m proximity fencing.
3. Passively scan BLE tags on equipment (stethoscopes, projectors) every 30 s.
4. Survive Wi-Fi and power loss by buffering to MicroSD and syncing later.
5. Obey the backend over MQTT (arm, disarm, reboot, logs, OTA) and report health by heartbeat.

The kiosk **does no fraud reasoning**. Verification (device, person, session, TOTP, enrollment, 11 SPARQL rules) happens server-side. The firmware's job is to capture faithfully, never lose or duplicate data, and show the verdict.

## 2. Hardware

| Part | Spec | Purpose |
| --- | --- | --- |
| ESP32-CAM | AI-Thinker, OV2640, dual-core, Wi-Fi + BLE | Capture, decode, networking |
| Storage | 4GB MicroSD, FAT32 | Offline scan buffer |
| Power | 5V 2A adapter + 18650 + 5V UPS shield | Mains with auto-switch to battery (6–8 h) |
| Feedback | Active piezo buzzer, green and red 5 mm LEDs | Scan verdict |
| Housing | Acrylic stand, fixed 10–12 cm | Constant focus distance, less glare |


### Pin and peripheral plan

The ESP32-CAM exposes very few GPIOs, and several are taken by the camera, SD card and flash.

- Run the SD card in **1-bit SD_MMC mode**. This frees GPIO4 so the flash LED does not flicker during SD access.
- Free pins after that: **GPIO 12, 13, 1, 3**.
- Suggested assignment: buzzer on 13, green LED on 12, red LED on 1, and a digital "on-battery / power-good" input from the UPS shield on 3.
- GPIO12 is a strapping pin and must be low at boot. An LED to ground with a resistor is safe.
- UART0 is repurposed, so production builds must disable serial logging.
- Add a **470 µF capacitor** near the 5V pin. The ESP32-CAM browns out easily when Wi-Fi TX coincides with the flash LED.
- The camera lens is manually adjustable. It must be **focused at \~10–12 cm** during assembly. Factory focus is far-field.

### Gaps in the Bill of Materials

- **No display**, yet MET says "student sees scan accepted on kiosk display". Feedback is only LEDs and buzzer.
- **No external RTC**, yet "RTC deep sleep" is specified. The internal RTC timer can wake the chip, but its time is lost on a full power loss (see §4.8).
- **No ADC** is available for a battery percentage (see §4.6).

## 3. Firmware architecture

### Core and task split 

- **Core 1 (high priority):** camera capture, QR decode, scan state machine, LED and buzzer driver. Nothing here may block on network or SD.
- **Core 0 (low priority):** Wi-Fi/BLE stacks (pinned there by default), MQTT client, HTTP uplink, SD sync worker, heartbeat, BLE asset scanner, OTA.
- Core 1 hands scans to Core 0 through a **FreeRTOS queue**. A dedicated writer task appends them to SD.

### Kiosk state machine

```
BOOT -> CONFIG(NVS) -> NET_UP (Wi-Fi, NTP, MQTT)
   -> DISARMED (BLE beacon on, camera idle, heartbeat running)
   -> ARMED    (camera active, scanning, sync paused)
   -> DISARMED (session ended)
   -> PRE_SLEEP (6:00 PM) -> DEEP_SLEEP -> wake 7:45 AM -> BOOT
   any state -> OTA / REBOOT on command
```

Persistent configuration in NVS: Wi-Fi credentials, MQTT and API endpoints, room ID, per-device secret, sleep and wake times, timezone.

## 4. Features and how they work

### 4.1 QR capture and decode

- The student holds their phone at 10–12 cm against the acrylic stand.
- The firmware extracts `{ studentId, totpHash, deviceUUID, timestamp }` and wraps it with the kiosk MAC, room ID, its own timestamp and the active session ID.
- **Camera setup:** grayscale, VGA or SVGA, decoded with a quirc-style library. Let auto-exposure converge on the bright phone screen.
- **Payload size matters.** A UUID plus hash plus ID makes a dense QR (version 7–10). Shortening the payload (compact binary or base45, truncated HMAC) lowers the version and speeds up decoding. \[Risk\]
- **Flash:**  "glare-free illumination". A phone screen is already emissive, and a flash on glass can cause specular glare. Default the flash **off or low-duty PWM**, and bench-test both. \[Risk\]
- **One scan equals one event.** The camera sees the same QR for many frames. Hash the payload and ignore repeats within about 3–5 s. \[Proposed\]
- Malformed or oversized payloads are dropped locally and get a reject signal.

### 4.2 Remote control channel (MQTT) 

- On session start the backend publishes to `room/{roomId}/status`: `{"armed": true, "duration": 600}`.
- **ARM:** the kiosk activates the camera, beeps once, and pauses SD sync.
- **DISARM** (timer end or teacher ends it): `{"armed": false}`. The kiosk flashes red and stops scanning.
- Other commands (from the Hardware Fleet page): remote reboot, log fetch, OTA firmware push.
- **Only armed kiosks scan.** This stops late scans, because the teacher's button defines the window.
- Hardening \[Proposed\]:
  - Use QoS 1 and a **retained** arm message, so a kiosk that reconnects mid-session picks up the correct state.
  - Send `sessionId`, `mode` (Timed or Continuous) and an **absolute `endsAt` epoch** instead of a relative duration. A reconnecting kiosk then knows the true remaining time.
  - Use an MQTT Last Will so the broker marks the kiosk offline immediately.
- **Extend (+2 min)** is a fresh arm message with a later `endsAt`. **Void** does not touch the kiosk, because it is a backend status change.

### 4.3 Verdict handling and feedback \[Spec\]

The kiosk POSTs the scan and waits for a verdict.

| Backend result | LED | Buzzer | Meaning |
| --- | --- | --- | --- |
| Verified | Green flash | 1 short beep | Clean scan |
| **Flagged** (Steps 5–6) | **Green flash** | **1 short beep** | **Honeypot**: the person must not learn they were caught |
| Rejected (Steps 1–4) | Red flash | 2 harsh beeps | Bad device, unknown or inactive person, no active session, expired or replayed QR |

Firmware rules:

- The kiosk must **never** distinguish Verified from Flagged in any local indication or log visible to the public.
- A request timeout is not a rejection. The scan is buffered to SD and gets the "pending" signal below.

**Offline "pending" feedback \[Proposed\].** With no verdict available, using green would be dishonest and red would cause panic re-scans. Use a distinct pattern, for example one short beep with alternating green and red LEDs. Then a student knows the scan was captured, though not yet verified. Teachers should be told that offline scans appear after sync.

### 4.4 BLE: two independent roles \[Spec\]

**A. Proximity beacon (anti-bunking).**

- The kiosk continuously advertises a BLE beacon.
- The student app renders the QR **only if the beacon's RSSI is within about 5 m (\~16 ft)**. A student in the corridor, or on a video call at home, never gets a QR.
- *Weakness* \[Risk\]: the check runs inside the app, so a modified or rooted app can skip it, and RSSI is a noisy distance estimate. *Hardening* \[Proposed\]: make the beacon carry a **rotating value** derived from a per-kiosk key. The app must embed the last-heard value in its QR payload, and the kiosk or server verifies it. A QR can then only be made by someone who recently heard the beacon.

**B. Passive asset scanning.**

- The same radio acts as a BLE observer, picking up tags on movable equipment.
- Every **30 s**, a telemetry payload of detected asset IDs goes to the semantic engine. Rule 11 compares the room against `:assignedToRoom` and flags displacement or theft.
- Practical handling \[Proposed\]:
  - Include RSSI and last-seen time, and ignore weak signals. BLE passes through walls, so neighbouring-room tags would otherwise be reported.
  - Send **deltas** (appeared and disappeared) with an occasional full snapshot, rather than the full list every time.
  - Use short scan windows with a duty cycle. Advertising, scanning and Wi-Fi share one radio, so unrestricted scanning hurts Wi-Fi throughput and scan latency.
  - Telemetry is lower priority than scans. If the buffer is full, drop old telemetry first.

### 4.5 Offline buffer and sync \[Spec; format Proposed\]

**Writing.** When Wi-Fi or the API is unreachable, each scan is appended as a CSV row on the MicroSD with the full payload and timestamp.

**Suggested record:** `seq, kioskMac, roomId, sessionId, kioskEpochMs, timeValid, studentId, totpHash, deviceUUID, qrTimestamp`

**Power-loss safety** \[Proposed\]:

- Append one line per scan and flush after each write.
- Keep a small `cursor` file recording the last acknowledged `seq`.
- Replaying a few rows after a crash is harmless because the backend deduplicates.

**Syncing.**

- A Core 0 worker starts only when Wi-Fi is back **and the scanner has been idle for 3–5 s continuously**.
- It reads **25–50 rows** per batch and POSTs to `session-batch-sync`, then advances the cursor after the server's ACK.
- Any new scan during sync pauses the worker immediately, so live scanning has priority.
- Use exponential backoff on failures.

**Server-side deduplication.** A compound unique index on `(sessionId + studentId)` with `bulkWrite({ordered:false})` silently drops duplicates without failing the batch. This relies on every row carrying the right `sessionId`, so the kiosk must stamp the session it received at arm time.

### 4.6 Heartbeat and health \[Spec\]

- Every **60 s**: `POST /devices/heartbeat` with MAC, battery %, firmware version and timestamp.
- The backend updates `:lastHeartbeat` and `:hasStatus` in the graph. A missed-heartbeat threshold flips the device to **Offline**, which feeds Rule 7 (tamper).
- **Battery %** \[Risk\]: this needs an ADC, and ADC2 pins cannot be used while Wi-Fi is on. The ESP32-CAM has no usable ADC1 pins. Realistic options: report only an "on battery / power-good" bit (§2 pin plan), or add an I²C fuel gauge or external ADC.
- **Pre-sleep notice** \[Proposed\]: before deep sleep, send a "sleeping until 07:45" heartbeat so the backend marks the kiosk *Sleeping* rather than Offline. Otherwise every night looks like a failure.

### 4.7 Power management \[Spec\]

- **Mains loss:** the UPS shield switches to the 18650 automatically. The kiosk should see no reset. It keeps scanning for about 6–8 hours.
- **Deep sleep from 6:00 PM to 7:45 AM:** the RTC timer wakes the chip. Draw is in micro-amps. Sleep duration is computed from NTP-synced time just before sleeping.
- **Wake sequence:** boot, reconnect Wi-Fi, re-sync NTP, reconnect MQTT, restore the retained arm state.
- Internal RTC drift is the reason to re-sync on every wake.
- In deep sleep the kiosk cannot hear MQTT. This is fine only if no sessions are scheduled in that window. Evening classes or hospital night shifts need a different sleep schedule per tenant, so keep sleep and wake times in NVS and updateable remotely \[Proposed\].

### 4.8 Timekeeping \[Proposed\]

- Every scan carries `kioskEpochMs`. It must be trustworthy because the backend and Rule 1 depend on it.
- NTP sync on boot and every few hours.
- If the kiosk reboots while offline and has no valid time, estimate from the last time persisted to SD (write it every minute) plus uptime, and set `timeValid=false` on those rows so the backend can treat them with care.
- An external RTC module (DS3231) would remove this class of problem for a small extra cost.

### 4.9 Security \[Proposed, since the Spec relies on MAC address alone\]

- **MAC addresses are trivially spoofable.** Pipeline Step 1 trusts them. Give each kiosk a per-device secret at registration (stored in NVS, with flash encryption if possible), and sign every request and batch with an HMAC.
- Use HTTPS for REST and TLS for MQTT, with per-device MQTT credentials and topic ACLs so one kiosk cannot publish to another room's topic.
- OTA images must be verified (hash and signature) before applying.
- Do not expose test endpoints. Disable serial in production.

### 4.10 Remote maintenance \[Spec; mechanics Proposed\]

- **OTA:** dual app partitions with automatic rollback. Only apply while disarmed. Mark the image valid only after a successful boot and heartbeat.
- **Remote log fetch:** a ring buffer on SD, uploaded on request.
- **Remote reboot:** only when disarmed, or after confirmation if armed.

---

## 5. Edge-case matrix

| # | Case | Risk | Handling |
| --- | --- | --- | --- |
| 1 | Campus Wi-Fi down | Scans never reach the server | Append to SD CSV, "pending" feedback, background batch sync on recovery with idle gating, deduplication via unique index (§4.5) |
| 2 | Wi-Fi down when the session should **start** | MQTT arm message cannot arrive, so the kiosk stays disarmed \[gap in Spec\] | Cache the room's schedule on SD and **self-arm** during scheduled slots in "continuous" mode when MQTT is unreachable. The teacher's later sync reconciles it. \[Proposed\] |
| 3 | Load shedding / power cut | Instant shutdown | UPS shield and 18650 switch automatically for 6–8 h. SD writes are flushed per line, so at most the in-flight scan is lost |
| 4 | Power cut then reboot | Time lost, buffered data in use | Resume from the SD cursor, estimate time from the last persisted value, flag `timeValid=false` (§4.8) |
| 5 | Screenshot sent over WhatsApp | Proxy attendance | QR regenerates every 10 s with a dynamic salt. Single-use nonce in Redis is consumed on the first scan |
| 5b | Same, but kiosk **offline** | The server cannot check the nonce until sync, and by then the 10 s window is long gone \[gap in Spec\] | Make the token verifiable offline: HMAC over `(studentId, deviceKey, 10 s time window)`. The kiosk rejects stale timestamps locally, and the server re-verifies and dedupes on sync. Single-use enforcement then applies at sync time |
| 6 | Two accounts on one phone | One phone, two students | Device UUID bound on first login and compared at Step 4. Second attempt is honeypotted. Prefer a hardware-backed key pair over IMEI (Android 10+ blocks IMEI access) |
| 7 | Camera glare and focal blur | Slow or failed scans | Fixed 10–12 cm stand, hand-focused lens, flash off or dimmed, grayscale capture, and a compact QR payload (§4.1) |
| 8 | Teacher delay / room change | Session not started or wrong room | Teacher claims any room's kiosk from the portal. The old session freezes and the new one arms on the claimed device. The kiosk is just re-armed over MQTT with a new `sessionId` |
| 9 | Night battery drain | Wasted battery | Deep sleep 6:00 PM to 7:45 AM, with pre-sleep notice (§4.6–4.7) |
| 10 | Lost or broken phone | Student cannot scan | Host manual override immediately. HOD resets the UUID binding via university-email OTP. The kiosk is not involved |
| 11 | Class cancelled after scanning | False presence | Teacher or HOD voids the session. It is marked CANCELLED, the kiosk is disarmed, and the records are excluded from percentages |
| 12 | Remote video-call proxy | QR shared from outside the room | BLE proximity fencing within about 5 m, with the hardening in §4.4A |
| 13 | Class moved to a room with no kiosk | No hardware | Teacher's Manual Roster mode with a local cache, synced later. No kiosk involvement |
| 14 | Scans during batch sync | New scans collide with an old batch | Core 1 has priority. The worker runs only after 3–5 s of idle and pauses on any new scan |
| 15 | Duplicate rows from SD | 100 records re-sent after a crash | Cursor file limits replays. Backend unique index and unordered `bulkWrite` discard the rest |
| 16 | Kiosk tampered with or removed | Manual false entries | Missed heartbeats set Offline. **Rule 7** (offline kiosk plus active session) raises a tamper flag and alerts the admin, who can switch to manual mode |
| 16b | Same, but only Wi-Fi was down | Rule 7 gives a **false tamper alert** \[gap\] | After reconnect, the kiosk sends an offline-interval report (down-since, buffered count, mains or battery) so the backend can clear the flag |
| 17 | Kiosk moved to another room | Fraudulent scans in the wrong place | Every scan carries the room ID from NVS. **Rule 8** compares it with `:registeredAt` |
| 18 | Too many scans too fast | Spoofed or automated scanning | Local debounce (§4.1). **Rule 5** counts distinct people per window on the server |
| 19 | Fraud detected | Offender must not know | Verdict handling shows green for Flagged (§4.3) |
| 20 | Asset leaves its room | Equipment theft | BLE observer reports to the semantic engine. **Rule 11** compares with `:assignedToRoom` |
| 21 | Beacon relay or spoofing | Fake proximity | Rotating beacon proof embedded in the QR (§4.4A) |
| 22 | Brownout / hang | Kiosk stops responding | Hardware watchdog on all tasks, a 470 µF capacitor, and a clean reboot that restores state from NVS and SD |
| 23 | Corrupt SD / full SD | Scan loss | Check at boot. If bad, switch to RAM ring buffer and send an alert in the heartbeat. Rotate and delete acknowledged segments |
| 24 | OTA fails mid-way | Bricked device | Dual partitions with rollback |

## 6. Backend assumptions that affect firmware

1. **Endpoint ownership differs.** AMS sends scans to the NestJS gateway, while MET sends them to Ali's semantic endpoint. Keep the base URL in NVS and configurable.
2. **Event time versus ingest time.** Rules like Rule 5 use `NOW()`. A synced batch of 100 buffered scans must be evaluated by the *kiosk event timestamp*, or it will trigger false proxy-device flags. Mark rows `source=buffered`.
3. **`sessionId` must be in every scan row.** The documented QR payload does not contain it, and the unique index needs it.
4. **Rule 5 threshold:** 15 distinct people in 3 minutes is about 5 per minute. A 50-person class in a 10-minute window reaches that normally, so tune it or it will fire constantly.
5. **Verdict for offline rows.** Rejections found during sync cannot be shown on the kiosk any more. They go to the admin review panel instead.

## 7. Suggested bench validation order

1. Focus and decode rate at 10–12 cm, with flash off, dimmed and on.
2. SD 1-bit mode with LEDs, buzzer and camera running together (pin plan).
3. Power-pull test during SD writes (cursor and duplicate behaviour).
4. Wi-Fi drop mid-session, then recovery (sync gating and priority).
5. BLE advertise, scan and Wi-Fi together under load.
6. Deep-sleep wake-time accuracy over several nights, and the retained arm state on reconnect.
7. OTA, including a deliberately bad image to prove rollback.
