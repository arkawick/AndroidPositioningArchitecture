# Android Location Architecture
## How Your Phone Determines Your Position
### Android + Qualcomm + SUPL Complete System

---

# Agenda

1. Introduction - The Location Challenge
2. System Architecture Overview
3. Android AOSP Stack
4. Qualcomm Hardware Architecture
5. SUPL Protocol Deep Dive
6. Complete Location Request Flow
7. Position Calculation & Mathematics
8. Performance & Real-World Metrics
9. Summary & Key Takeaways

---

# The Location Challenge

## Why is GPS Positioning Hard?

**Traditional GPS Problems:**
- **Slow Time To First Fix (TTFF):** 30-60+ seconds cold start
- **Weak Signals:** -160 dBm (weaker than a 25W bulb on the Moon)
- **Data Download:** 12.5-30 minutes for almanac/ephemeris from satellites
- **High Power Consumption:** Extended satellite search drains battery
- **Poor Indoor Performance:** Signals blocked by buildings

**Result:** Bad user experience without assistance!

---

# The Solution: A-GPS with SUPL

## Assisted GPS (A-GPS)

**Key Concept:** Download assistance data over fast internet instead of slow satellite signals

**Speed Comparison:**
- Satellite broadcast: 50 bits/second
- 4G/5G internet: 20+ Mbps (400,000× faster!)

**Result:**
- TTFF: 5-10 seconds (vs 30-60 seconds)
- 90% less power consumption
- Works in weak signal environments

---

# Complete System Overview

## Three Major Components

```
┌─────────────────────────────────────────┐
│         Android AOSP Stack              │
│  (Application → Framework → HAL)        │
└─────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────┐
│      Qualcomm Hardware & Firmware       │
│  (SoC → Modem → Hexagon DSP → GNSS)    │
└─────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────┐
│           SUPL Network                  │
│  (Assistance Data Servers)              │
└─────────────────────────────────────────┘
```

**Plus:** GPS/GLONASS/Galileo/BeiDou satellites orbiting at 20,000 km

---

# Android AOSP Stack
## 7-Layer Architecture

**Layer 1: Applications**
- Google Maps, Uber, Weather apps
- Use LocationManager API

**Layer 2: Android Framework (Java)**
- LocationManagerService (system service)
- GnssLocationProvider (GPS provider)
- Permission enforcement, provider selection

**Layer 3: JNI Bridge (C++)**
- android_location_GnssLocationProvider.cpp
- Java ↔ Native code translation

---

# Android AOSP Stack (cont.)

**Layer 4: GNSS HAL (HIDL/AIDL)**
- IGnss, IGnssCallback interfaces
- Hardware abstraction layer
- Vendor-independent API

**Layer 5: Vendor HAL Implementation**
- Qualcomm's Gnss.cpp
- Location API (Qualcomm proprietary)
- GNSS daemon (gnss@2.1-service)

**Layer 6: QMI Protocol**
- Communication to modem subsystem
- Message-based protocol (TLV encoding)

**Layer 7: GNSS Engine**
- Runs on Hexagon DSP / Modem processor

---

# Qualcomm Hardware Architecture

## Snapdragon SoC Components

**1. Application Processor (Kryo CPU)**
- Runs Android OS
- ARM Cortex-based cores
- Handles apps and framework

**2. Modem Subsystem**
- Separate processor complex
- Handles cellular (4G/5G) communications
- Has its own CPU, memory, and OS

**3. Hexagon DSP**
- Digital Signal Processor
- Low-power, specialized processor
- Runs GNSS engine firmware

---

# GNSS Engine Architecture

## Running on Hexagon DSP

**Components:**

1. **RF Frontend**
   - Antenna and Low Noise Amplifier (LNA)
   - RF to baseband conversion
   - L1/L5 frequency reception (1.5 GHz)

2. **Acquisition Engine**
   - Searches for satellite signals
   - Parallel correlators
   - Doppler frequency search

3. **Tracking Channels** (12-32 channels)
   - Code tracking loop (DLL)
   - Carrier tracking loop (PLL)
   - One channel per satellite

---

# GNSS Engine Architecture (cont.)

4. **Measurement Engine**
   - Generates pseudorange measurements
   - Doppler shift measurements
   - Carrier phase measurements
   - Signal strength (C/N0)

5. **Position Engine**
   - Trilateration algorithm
   - Least squares solution
   - Kalman filtering
   - ECEF → Lat/Lon/Alt conversion

6. **SUPL Client**
   - Downloads assistance data
   - Manages SUPL sessions

---

# QMI Protocol

## Qualcomm MSM Interface

**Purpose:** Communication between Application Processor and Modem/GNSS

**Architecture:**
- Service-based (QMI_LOC for location)
- Message-based protocol
- TLV (Type-Length-Value) encoding

**Key Messages:**
- `QMI_LOC_START_REQ`: Start GPS tracking
- `QMI_LOC_STOP_REQ`: Stop GPS
- `QMI_LOC_POSITION_REPORT_IND`: Position update
- `QMI_LOC_INJECT_SUPL_*`: Assistance data injection

**Transport:** IPC Router (modern) or SMD (legacy)

---

# SUPL Protocol Overview

## Secure User Plane Location

**Developed by:** Open Mobile Alliance (OMA)
- SUPL 1.0 (2007)
- SUPL 2.0 (2012)

**Key Innovation:** Use IP network (user plane) instead of cellular control plane

**Benefits:**
- Network-independent (works over WiFi, 4G, 5G)
- Simpler deployment
- Lower infrastructure cost
- Better roaming support
- Single standard for all networks

---

# SUPL Architecture

## Two Main Components

**1. SET (SUPL Enabled Terminal)**
- Your mobile device
- Contains SUPL client software
- GPS/GNSS receiver
- IP connectivity

**2. SLP (SUPL Location Platform)**
- Network server (e.g., supl.google.com)
- Provides assistance data
- Can calculate position (SET-Assisted mode)

**SLP consists of:**
- **SLC** (SUPL Location Center): Session management
- **SPC** (SUPL Positioning Center): Position calculation

---

# SUPL Session Flow

## Network-Initiated (Proxy Mode)

```
1. LCS Client → SLP: Location request (MLP)
2. SLP → SET: SUPL INIT (trigger)
3. SET → SLP: TLS connection established
4. SET → SLP: SUPL POS INIT (capabilities)
5. SLP → SET: SUPL POS (assistance data)
6. SET ↔ SLP: SUPL POS (positioning protocol)
7. SLP → SET: SUPL END (position result)
8. SLP → LCS Client: Location response
```

**Use Case:** Emergency services, fleet tracking

---

# SUPL Session Flow (cont.)

## SET-Initiated (Non-Proxy Mode)

```
1. App → SET: Request location
2. SET → SLP: SUPL START (request assistance)
3. SLP → SET: SUPL RESPONSE (acknowledge)
4. SET → SLP: SUPL POS INIT (request data)
5. SLP → SET: SUPL POS (assistance data)
6. SET: Calculate position locally
7. SET → SLP: SUPL END (session complete)
8. SET → App: Location delivered
```

**Use Case:** Navigation apps, maps

---

# Assistance Data

## What SUPL Provides

**1. Reference Time**
- Accurate GPS time
- Eliminates 30+ second download

**2. Reference Location**
- Approximate position from cell tower
- Reduces satellite search space

**3. Satellite Ephemeris**
- Precise orbital data for each satellite
- Valid ~4 hours, 72 bytes per satellite
- Eliminates 30-second download

**4. Satellite Almanac**
- Coarse orbital data for all satellites
- Valid ~6 months, 900 bytes total

---

# Assistance Data (cont.)

**5. Ionospheric Model**
- Correction for atmospheric delay
- Improves accuracy by 10-20 meters

**6. UTC Model**
- GPS time ↔ UTC conversion

**7. Acquisition Assistance**
- Expected Doppler frequency for each satellite
- Expected code phase
- Search window parameters
- **Most critical for fast TTFF!**

**Total data size:** ~12 KB per session

---

# Position Calculation Methods

## Two Modes

### SET-Based (MS-Based)
**"Device measures and calculates"**
- SLP provides assistance data
- SET acquires satellites
- SET calculates position locally
- Better privacy, works offline after assistance
- **Preferred by Android devices**

### SET-Assisted (MS-Assisted)
**"Device measures, network calculates"**
- SLP provides assistance data
- SET makes GPS measurements
- SET sends measurements to SLP
- SLP calculates and returns position
- Better accuracy, lower device power

---

# GPS Measurement Fundamentals

## How GPS Works

**Basic Principle:** Time-of-Flight measurement

1. Satellite broadcasts signal with timestamp
2. Receiver measures signal arrival time
3. Calculate distance: `distance = speed_of_light × time_delay`
4. This is called a "pseudorange" (includes clock error)

**Why 4 satellites minimum?**
- 3 satellites → 3D position (X, Y, Z)
- 4th satellite → Solve receiver clock bias
- More satellites → Better accuracy (overdetermined system)

---

# Navigation Equations

## The Mathematics

**For each satellite i:**

```
ρᵢ = √[(Xᵢ - X)² + (Yᵢ - Y)² + (Zᵢ - Z)²] + c·Δt
```

Where:
- `ρᵢ` = Measured pseudorange to satellite i
- `(Xᵢ, Yᵢ, Zᵢ)` = Satellite position (from ephemeris)
- `(X, Y, Z)` = Receiver position (unknown)
- `c` = Speed of light (299,792,458 m/s)
- `Δt` = Receiver clock bias (unknown)

**4 unknowns:** X, Y, Z, Δt
**4+ equations:** One per satellite
**Solution:** Least squares minimization

---

# Position Calculation Process

## Iterative Solution

**Step 1:** Initial guess (from SUPL reference location)
```
X₀ = -2,707,000 m  (approx 37.42°N, 122.08°W)
Y₀ = -4,260,000 m
Z₀ = 3,870,000 m
Δt₀ = 0.001 s
```

**Step 2:** Linearize equations using Taylor series

**Step 3:** Form matrix equation
```
Δρ = H · Δx
```
Where H is the observation matrix (N satellites × 4 unknowns)

---

# Position Calculation (cont.)

**Step 4:** Solve using weighted least squares
```
Δx = (H^T · W · H)^(-1) · H^T · W · Δρ
```
Weight matrix W favors:
- Stronger signals (higher C/N0)
- Higher elevation satellites
- Lower multipath

**Step 5:** Update and iterate
```
X₁ = X₀ + ΔX
Y₁ = Y₀ + ΔY
Z₁ = Z₀ + ΔZ
Δt₁ = Δt₀ + ΔΔt
```

**Repeat** until convergence (typically 3-5 iterations)

---

# Kalman Filtering

## Smoothing the Solution

**Purpose:** Combine multiple measurements over time for better accuracy

**State Vector:**
```
X = [position_x, position_y, position_z,
     velocity_x, velocity_y, velocity_z,
     clock_bias, clock_drift]
```

**Process:**
1. **Predict:** Estimate next state based on motion model
2. **Update:** Correct prediction with new GPS measurements
3. **Iterate:** Repeat every measurement epoch (1 Hz)

**Benefits:**
- Smoother position estimates
- Velocity estimation
- Handles noisy measurements
- Predicts during brief signal loss

---

# Error Corrections

## Sources of Error

| Error Source | Magnitude | Correction |
|--------------|-----------|------------|
| Satellite clock | ±2 m | Ephemeris parameters |
| Ionospheric delay | 5-15 m | Ionospheric model |
| Tropospheric delay | 2-3 m | Mathematical model |
| Multipath | 0-10 m | Signal processing |
| Receiver noise | 1-3 m | Filtering |
| Ephemeris error | 1-5 m | Differential GPS |

**Total Accuracy:**
- A-GPS: 5-10 meters (95% confidence)
- Standalone GPS: 10-30 meters

---

# Multi-Constellation GNSS

## Beyond GPS

**SUPL 2.0 supports multiple satellite systems:**

| System | Country | Satellites | Status |
|--------|---------|------------|--------|
| GPS | USA | 31 | Operational |
| GLONASS | Russia | 24 | Operational |
| Galileo | EU | 30 | Operational |
| BeiDou | China | 49 | Operational |
| QZSS | Japan | 7 | Regional |
| NavIC | India | 7 | Regional |

**Benefits:**
- More visible satellites (12+ typically)
- Better geometry → lower DOP
- Improved accuracy: 3-5 meters
- Faster TTFF
- Better urban canyon performance

---

# Complete Flow: Real-World Example

## Opening Google Maps

**Location:** Google HQ (37.422°N, 122.084°W)
**Device:** Snapdragon 888 phone
**Network:** 4G LTE

Let's trace the complete 3.3-second journey...

---

# Timeline: T=0.000s to T=0.250s

**T=0.000s:** User taps Maps icon
- Android launches application
- Maps requests high-accuracy location

**T=0.050s:** LocationManagerService
- Permission check: ACCESS_FINE_LOCATION ✓
- Provider selection: GnssLocationProvider
- Call enable() and start()

**T=0.100s:** GNSS HAL initialization
- JNI call to native code
- Load Qualcomm HAL (IGnss interface)
- Register callbacks

**T=0.150s:** Configure position mode
- Mode: MS_BASED
- Recurrence: PERIODIC
- Interval: 1000 ms

**T=0.200s:** Inject assistance
- Time: Network time ±50ms
- Location: Last WiFi position ±500m

**T=0.250s:** Start GPS
- IGnss.start() called
- QMI_LOC_START_REQ sent to GNSS engine

---

# Timeline: T=0.300s to T=1.100s

**T=0.300s:** GNSS engine powers up
- RF frontend ON
- Tracking channels initialized
- Response: SUCCESS

**T=0.350s:** Trigger SUPL session
- Check: Need assistance (no ephemeris)

**T=0.400s:** DNS lookup
- Resolve supl.google.com → 172.217.15.174

**T=0.450s:** TCP connection
- Connect to port 7275
- 3-way handshake (20ms RTT)

**T=0.500s:** TLS handshake
- Certificate exchange
- Secure connection established (50ms)

**T=0.600s:** SUPL START
- Send capabilities, cell ID
- Message: ~500 bytes

**T=0.650s:** SUPL RESPONSE
- SLP acknowledges, confirms MS_BASED mode

**T=0.700s:** SUPL POS INIT
- Request ephemeris, almanac, acquisition assistance

**T=0.800s:** Server generates assistance
- Calculate visible satellites
- Retrieve ephemeris from reference network
- Calculate Doppler and code phase

**T=1.100s:** SUPL POS - Assistance delivery
- GPS: 10 satellites
- GLONASS: 6 satellites
- Galileo: 4 satellites
- BeiDou: 4 satellites
- Total: ~12 KB downloaded

---

# Timeline: T=1.400s to T=3.100s

**T=1.400s:** GNSS engine receives assistance
- Parse and store data
- Load acquisition parameters

**T=1.450s:** Satellite acquisition begins
- Parallel search using assistance data

**T=1.500s:** First satellite acquired (PRN 5)
- Expected: -2534 Hz, code 512
- Found: -2531 Hz, code 514 ✓

**T=1.600s - T=2.500s:** More satellites acquired
- PRN 7, 13, 15, 18, 21, 24, 30
- Plus GLONASS and Galileo satellites
- **Total: 12 satellites tracked**

**T=3.000s:** First measurement epoch
- 12 pseudoranges
- 12 Doppler measurements
- 12 carrier phases
- 12 C/N0 values

**T=3.050s:** Position calculation
- Load satellite positions
- Apply corrections
- Least squares (3-4 iterations)
- Kalman filter initialization
- ECEF → Lat/Lon/Alt

**Calculated Position:**
- Lat: 37.422401°N (error: -0.7m)
- Lon: -122.084063°W (error: +0.5m)
- Alt: 31.8 m
- Accuracy: ±4.2m (HDOP: 0.9)
- **Computation time: 50ms**

**T=3.100s:** Position reported to Android
- QMI_LOC_POSITION_REPORT_IND
- GNSS daemon → Location API → HAL
- IGnssCallback.gnssLocationCb()
- JNI → GnssLocationProvider

---

# Timeline: T=3.150s to Done

**T=3.150s:** Framework processes location
- GnssLocationProvider.handleReportLocation()
- LocationManagerService.handleLocationChanged()
- Create Location object

**T=3.200s:** Location delivered to Maps
- LocationListener callback
- Executed on Maps' main thread

**T=3.250s:** Maps updates UI
- **Blue dot appears!**
- Map centers on your location
- "Your location" marker

**T=3.300s:** SUPL session terminates
- SUPL END sent
- TLS connection closed

**T=4.000s onwards:** Subsequent fixes
- Every 1 second: new position
- No SUPL needed (ephemeris valid 4 hours)
- Each fix: ~100ms measurement → callback

---

# Performance Metrics

## Time To First Fix (TTFF)

| Scenario | Standalone GPS | SUPL A-GPS | Improvement |
|----------|---------------|------------|-------------|
| Cold Start | 30-60 sec | 5-10 sec | **6-10× faster** |
| Warm Start | 20-30 sec | 3-5 sec | **6-8× faster** |
| Hot Start | 5-10 sec | 1-3 sec | **3-5× faster** |

**Cold:** No almanac/ephemeris/time (device off >4 hours)
**Warm:** Has almanac, stale ephemeris (off 1-4 hours)
**Hot:** Recent data (off <1 hour)

---

# Performance Metrics (cont.)

## Accuracy Comparison

| Method | Typical | Ideal | Poor Conditions |
|--------|---------|-------|-----------------|
| Standalone GPS | 10-30m | 5-10m | 30-100m |
| SUPL A-GPS | **5-10m** | **3-5m** | 10-30m |
| Multi-GNSS | **3-5m** | **2-3m** | 5-15m |

## Power Consumption

| Operation | Power | Duration | Energy |
|-----------|-------|----------|--------|
| Standalone GPS | 150-200 mW | 30-60s | 4.5-12 Ws |
| A-GPS | 150-200 mW | 5-10s | 0.75-2 Ws |
| **Savings** | - | - | **83-93%** |

---

# Data Usage

## SUPL Session

**Download (from SLP):**
- Ephemeris data: ~10 KB
- Almanac: ~1 KB
- Ionospheric model: ~0.5 KB
- Acquisition assistance: ~0.5 KB
- **Total: ~12 KB per session**

**Upload (to SLP):**
- SUPL START: ~0.5 KB
- SUPL POS INIT: ~0.3 KB
- SUPL END: ~0.2 KB
- **Total: ~1 KB per session**

**Frequency:**
- Initial session: 13 KB
- Refresh needed every 4 hours (ephemeris expiration)
- **Daily usage: ~75 KB** (assuming 6 position requests per day)

---

# DOP (Dilution of Precision)

## Geometric Quality

**What is DOP?**
Measure of how satellite geometry affects position accuracy

**Types:**
- **GDOP** (Geometric): Overall 3D + time
- **PDOP** (Position): 3D position only
- **HDOP** (Horizontal): Lat/Lon accuracy
- **VDOP** (Vertical): Altitude accuracy

**Values:**
- **1.0-2.0:** Excellent
- **2.0-5.0:** Good
- **5.0-10.0:** Moderate
- **>10.0:** Poor

**Impact:** Position error = Range error × DOP

---

# Security Features

## SUPL Security

**Transport Security:**
- TLS 1.2/1.3 encryption
- Certificate-based authentication
- Protects against eavesdropping

**Authentication Methods:**
- GBA (Generic Bootstrapping Architecture)
- PSK (Pre-Shared Key)
- Certificate-based

**Privacy Protection:**
- User notification for location requests
- Location consent mechanisms
- Opt-out capabilities

---

# Summary: The Complete Picture

## 13 Steps from App to Satellites and Back

1. Application requests location (LocationManager API)
2. Framework processes (LocationManagerService)
3. GNSS provider activated (GnssLocationProvider)
4. HAL called (IGnss interface)
5. Vendor implementation (Qualcomm Location API)
6. QMI message to modem (IPC Router)
7. GNSS engine on Hexagon DSP
8. SUPL session for assistance data
9. RF frontend receives satellite signals
10. Acquisition and tracking engines lock satellites
11. Position engine calculates location
12. QMI callback through all layers
13. Blue dot appears in Maps

**Total time: 3-5 seconds**

---

# Key Takeaways

## What Makes Modern Location Services Fast

1. **A-GPS with SUPL**
   - 10× faster than standalone GPS
   - 90% less power consumption
   - Works in weak signal environments

2. **Multi-Constellation GNSS**
   - GPS + GLONASS + Galileo + BeiDou
   - 12+ satellites tracked simultaneously
   - 3-5 meter accuracy

3. **Specialized Hardware**
   - Hexagon DSP for low-power processing
   - Parallel tracking channels
   - Real-time position calculation

4. **Efficient Software Architecture**
   - Clean separation: AOSP → HAL → Vendor
   - QMI protocol for IPC
   - Kalman filtering for smoothness

---

# Technology Integration

## The Magic of Seamless Location

**What happens when you tap Maps:**

- Android OS manages the request
- Qualcomm hardware processes signals from space
- SUPL network provides assistance over internet
- Satellites 20,000 km away provide timing
- Position calculated to within 5 meters
- Blue dot appears in 3 seconds

**All working together seamlessly!**

This is the result of decades of engineering across:
- Satellite technology
- Radio signal processing
- Network protocols
- Mobile operating systems
- Specialized hardware

---

# Glossary - Part 1

**A-GPS (Assisted GPS):** GPS with assistance data from network servers

**AOSP:** Android Open Source Project

**BeiDou:** China's satellite navigation system (49 satellites)

**C/N0:** Carrier-to-Noise density ratio, signal strength measure (dB-Hz)

**DOP (Dilution of Precision):** Geometric quality of satellite configuration

**DSP (Digital Signal Processor):** Specialized processor for signal processing

**ECEF:** Earth-Centered Earth-Fixed coordinate system

**Ephemeris:** Precise satellite orbital parameters (valid ~4 hours)

**Galileo:** European Union's satellite navigation system (30 satellites)

---

# Glossary - Part 2

**GLONASS:** Russia's satellite navigation system (24 satellites)

**GNSS (Global Navigation Satellite System):** Generic term for GPS, GLONASS, etc.

**GPS:** US Global Positioning System (31 satellites)

**HAL (Hardware Abstraction Layer):** Android's vendor-independent hardware interface

**HDOP:** Horizontal Dilution of Precision

**Hexagon DSP:** Qualcomm's proprietary DSP architecture

**IPC (Inter-Process Communication):** Communication between separate processes

**JNI (Java Native Interface):** Bridge between Java and C/C++ code

**Kalman Filter:** Algorithm for combining noisy measurements over time

---

# Glossary - Part 3

**LPP (LTE Positioning Protocol):** Positioning protocol for LTE networks

**MS-Assisted:** SET-Assisted mode (device sends measurements to server)

**MS-Based:** SET-Based mode (device calculates position)

**Pseudorange:** Distance measurement including clock error

**QMI (Qualcomm MSM Interface):** Message protocol for Qualcomm SoC communication

**RRLP:** Radio Resource LCS Protocol (for GSM)

**SET (SUPL Enabled Terminal):** Mobile device with SUPL client

**SLC (SUPL Location Center):** SUPL server session management component

**SLP (SUPL Location Platform):** SUPL server infrastructure

---

# Glossary - Part 4

**SPC (SUPL Positioning Center):** SUPL server position calculation component

**SUPL:** Secure User Plane Location protocol

**TLS (Transport Layer Security):** Encryption protocol

**Trilateration:** Position determination from distance measurements

**TTFF (Time To First Fix):** Time from GPS start to first position

**ULP (UserPlane Location Protocol):** Core SUPL messaging protocol

**VDOP:** Vertical Dilution of Precision

**WGS84:** World Geodetic System 1984 (GPS coordinate reference)

---

# Questions?

## Discussion Topics

- Specific implementation details
- Performance optimization
- Power consumption analysis
- Alternative positioning methods
- Future enhancements (5G positioning, etc.)

---

# References

## Documentation & Specifications

1. **Android Source:**
   - AOSP GNSS HAL: source.android.com/devices/sensors/gnss
   - LocationManager API docs

2. **SUPL Specifications:**
   - OMA SUPL 2.0 Architecture (OMA-AD-SUPL-V2_0)
   - Open Mobile Alliance: openmobilealliance.org

3. **Qualcomm:**
   - Location Suite documentation
   - Snapdragon platform specifications

4. **GPS/GNSS:**
   - GPS Interface Specification (IS-GPS-200)
   - GLONASS ICD, Galileo OS-SIS-ICD

---

# Thank You!

## Complete Android Location Architecture
### Android AOSP + Qualcomm Hardware + SUPL Protocol

**From tap to blue dot in 3 seconds**

---

**End of Presentation**
