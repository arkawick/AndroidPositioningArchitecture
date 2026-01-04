# Android, Qualcomm & SUPL Location Architecture
## Complete Technical Deep Dive: How Position is Determined

**Comprehensive Guide to Location Services in Android with Qualcomm Hardware**

---

## Table of Contents

**Part 1: Foundation (Slides 1-10)**
1. Introduction to Mobile Location Services
2. Complete Architecture Overview
3. Android Location Stack
4. Qualcomm SoC Architecture
5. Communication Path: App to Modem
6. GNSS Hardware Components
7. SUPL Introduction
8. Key Components Summary
9. Location Request Types
10. Operating Modes

**Part 2: Deep Dive (Slides 11-30)**
11. Complete Location Request Flow
12. Android Framework Layer
13. Location Manager Service
14. GNSS Location Provider
15. RIL Architecture
16. Radio HAL Interface
17. Qualcomm Vendor RIL
18. QMI Protocol
19. Modem Subsystem
20. GNSS Engine Architecture
21. SUPL Session Initiation
22. Assistance Data Delivery
23. GPS Measurements
24. Position Calculation Methods
25. SET-Assisted vs SET-Based
26. Navigation Mathematics
27. Error Corrections
28. Multi-GNSS Support
29. Performance Metrics
30. Summary & Conclusion

---

# PART 1: FOUNDATION

---

## Slide 1: Title Slide

**Android, Qualcomm & SUPL Location Architecture**

*Complete Technical Deep Dive: How Your Device Determines Position*

**Topics Covered:**
- Android AOSP Location Framework
- Qualcomm Modem & GNSS Integration
- SUPL (Secure User Plane Location)
- Complete Flow: Application → Satellite → Position

---

## Slide 2: Introduction - Why Location Matters

**Ubiquitous Location Services**

**Applications:**
- Navigation & Maps (Google Maps, Waze)
- Emergency Services (E911/E112)
- Ride Sharing (Uber, Lyft)
- Fitness & Health Tracking
- Geo-tagging & Social Media
- Asset Tracking & Fleet Management
- Augmented Reality

**Technical Challenges:**
- Fast Time To First Fix (TTFF)
- High accuracy (< 10 meters)
- Low power consumption
- Indoor/Urban canyon performance
- Privacy & security

---

## Slide 3: Complete Architecture Overview

**End-to-End Location System**

```
┌──────────────────────────────────────────────────────────┐
│  Layer 1: Applications                                   │
│  (Maps, Navigation, Emergency, Location-Based Services)  │
├──────────────────────────────────────────────────────────┤
│  Layer 2: Android Framework                              │
│  - LocationManager, FusedLocationProvider               │
│  - LocationManagerService                                │
├──────────────────────────────────────────────────────────┤
│  Layer 3: GNSS Location Provider                         │
│  - GnssLocationProvider (Java)                           │
│  - JNI Bridge (C++)                                      │
├──────────────────────────────────────────────────────────┤
│  Layer 4: GNSS HAL (Hardware Abstraction)                │
│  - IGnss, IGnssMeasurement, IGnssConfiguration           │
│  - HIDL/AIDL Interface                                   │
├──────────────────────────────────────────────────────────┤
│  Layer 5: Qualcomm Vendor Implementation                 │
│  - Location HAL, Location API                            │
│  - QMI Client                                            │
├──────────────────────────────────────────────────────────┤
│  Layer 6: GNSS Engine                                    │
│  - Position Engine, Measurement Engine                   │
│  - SUPL Client                                           │
├──────────────────────────────────────────────────────────┤
│  Layer 7: Hardware                                       │
│  - GNSS Chipset, RF Frontend                             │
│  - Multi-constellation receiver                          │
├──────────────────────────────────────────────────────────┤
│  External: SUPL Server (SLP)                             │
│  - Assistance data provider                              │
│  - Position calculation (SET-Assisted mode)              │
└──────────────────────────────────────────────────────────┘
          ↕
   GPS/GLONASS/Galileo/BeiDou Satellites
```

---

## Slide 4: Android Location Stack

**Framework Architecture**

**Key Components:**

**1. Application Layer**
```java
LocationManager locationManager = getSystemService(Context.LOCATION_SERVICE);
LocationRequest request = LocationRequest.create()
    .setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY)
    .setInterval(10000);
```

**2. LocationManagerService**
- Central system service in `system_server`
- Manages all location providers
- Enforces permissions (ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION)
- Handles client requests

**3. Location Providers**
- **GPS Provider**: GNSS-based positioning
- **Network Provider**: Cell + WiFi positioning
- **Fused Provider**: Google Play Services fusion algorithm
- **Passive Provider**: Listens to other apps' location updates

**4. GnssLocationProvider**
- Native GNSS implementation
- Interfaces with GNSS HAL
- Processes raw measurements
- Manages satellite data

---

## Slide 5: Qualcomm SoC Architecture

**Snapdragon Platform Components**

```
┌─────────────────────────────────────────────────────┐
│  Qualcomm Snapdragon SoC                            │
│                                                     │
│  ┌──────────────────────┐  ┌────────────────────┐ │
│  │ Application Processor │  │ Modem Subsystem    │ │
│  │ (AP)                  │  │ (MSS)              │ │
│  │                       │  │                    │ │
│  │ - ARM Cortex CPUs     │  │ - Modem Processor  │ │
│  │ - Android OS/Linux    │  │ - Protocol Stack   │ │
│  │ - Framework Services  │  │ - AMSS Firmware    │ │
│  │ - Applications        │  │ - QMI Services     │ │
│  └───────────┬───────────┘  └─────────┬──────────┘ │
│              │                        │            │
│              └────────IPC─────────────┘            │
│               (IPC Router/SMD)                     │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │ GNSS Engine (Hexagon DSP)                    │  │
│  │ - Position Engine (PE)                       │  │
│  │ - Measurement Engine (ME)                    │  │
│  │ - Kalman Filter                              │  │
│  │ - SUPL Client                                │  │
│  └──────────────────┬───────────────────────────┘  │
│                     │                              │
│  ┌──────────────────────────────────────────────┐  │
│  │ GNSS RF Frontend & Chipset                   │  │
│  │ - Multi-band receiver (L1/L5)                │  │
│  │ - 32+ tracking channels                      │  │
│  │ - GPS/GLONASS/Galileo/BeiDou                 │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Key Features:**
- Dedicated GNSS processor (offloaded from AP)
- Hardware-accelerated position calculation
- Low-power tracking modes
- Multi-constellation support

---

## Slide 6: Communication Path: App to Modem

**Complete Request Flow**

```
Application (Java)
    ↓ Binder IPC
LocationManagerService (system_server)
    ↓
GnssLocationProvider (Java)
    ↓ JNI call
android_location_GnssLocationProvider.cpp
    ↓ HIDL/AIDL
GNSS HAL Interface (IGnss)
    ↓
Qualcomm GNSS HAL Implementation (vendor)
    ↓
Location Integration API (loc_api)
    ↓
Location Client API
    ↓ LocIPC (socket)
GNSS Daemon (gnss@2.1-service)
    ↓
Location API
    ↓
Position Engine (PE)
    ↓
Measurement Engine (ME)
    ↓
GNSS Chipset Hardware
    ↓
RF Frontend
    ↓
Satellites (GPS/GLONASS/Galileo/BeiDou)
```

**Parallel Communication with SUPL:**
```
GNSS Engine ←→ SUPL Client ←→ Internet ←→ SUPL Server (SLP)
                     ↕
           Assistance Data Exchange
```

---

## Slide 7: GNSS Hardware Components

**Qualcomm GNSS Chipset Architecture**

**1. RF Frontend**
- Multi-band receiver: L1 (1575 MHz), L5 (1176 MHz)
- Low Noise Amplifier (LNA)
- Down-converter to baseband
- Analog-to-Digital Converter (ADC)

**2. Acquisition Engine**
- Parallel search for satellite signals
- PRN code correlation
- Doppler frequency detection
- Sensitivity: -148 dBm (acquisition), -160 dBm (tracking)

**3. Tracking Channels (32+)**
- GPS: 24 channels
- GLONASS: 8 channels
- Galileo: 8 channels
- BeiDou: 8 channels

**4. Correlators**
- Early/Prompt/Late correlators
- DLL (Delay Lock Loop) - tracks code phase
- PLL (Phase Lock Loop) - tracks carrier phase
- FLL (Frequency Lock Loop) - tracks carrier frequency

**5. Navigation Processor (Hexagon DSP)**
- Signal processing
- Measurement generation
- Position calculation
- Kalman filtering

---

## Slide 8: SUPL Introduction

**Secure User Plane Location Protocol**

**What is SUPL?**
- OMA (Open Mobile Alliance) standard
- IP-based location protocol using User Plane
- Provides Assisted GNSS (A-GNSS) data
- Faster TTFF and improved accuracy

**Key Principle:**
Use existing IP connectivity (cellular data/WiFi) to deliver GPS assistance data instead of downloading it slowly from satellites.

**Architecture:**

```
┌────────────────────────────────────────────┐
│  SET (SUPL Enabled Terminal)               │
│  - Mobile device                           │
│  - SUPL client                             │
│  - GPS/GNSS receiver                       │
└──────────────┬─────────────────────────────┘
               ↕ ULP over IP (TLS)
┌────────────────────────────────────────────┐
│  SLP (SUPL Location Platform)              │
│  ┌────────────────────────────────────┐   │
│  │ SLC (SUPL Location Center)         │   │
│  │ - Session management               │   │
│  │ - Authentication                   │   │
│  └────────────────────────────────────┘   │
│  ┌────────────────────────────────────┐   │
│  │ SPC (SUPL Positioning Center)      │   │
│  │ - Assistance data generation       │   │
│  │ - Position calculation             │   │
│  └────────────────────────────────────┘   │
└────────────────────────────────────────────┘
               ↕
   Reference Network (GPS monitoring stations)
```

---

## Slide 9: Location Request Types

**Priority Modes and Characteristics**

| Mode | Provider | Accuracy | Power | TTFF | Use Case |
|------|----------|----------|-------|------|----------|
| **HIGH_ACCURACY** | GPS/GNSS | 3-5m | 100-150mW | 5-10s | Navigation, Maps |
| **BALANCED_POWER** | Fused (GPS+Network) | 10-100m | 10-30mW | 2-5s | General apps |
| **LOW_POWER** | Network (Cell+WiFi) | 100-1000m | 1-5mW | 1-2s | Background tracking |
| **NO_POWER** | Passive listening | Variable | 0mW | 0s | Opportunistic |

**Additional Request Types:**
- **Single Shot**: One-time location request
- **Continuous Tracking**: Periodic updates (e.g., 1 Hz)
- **Batching**: Collect locations in hardware, deliver in batch (fitness tracking)
- **Geofencing**: Hardware-monitored geographical boundaries

---

## Slide 10: Operating Modes

**Location Calculation Methods**

**1. Standalone GPS**
- No assistance data
- Device downloads ephemeris from satellites (30s - 12.5 min)
- Full autonomous operation
- TTFF: 30-60 seconds (cold start)

**2. Assisted GPS (A-GPS) - SET-Based**
- SUPL provides assistance data
- Device calculates position locally
- Better privacy
- TTFF: 5-10 seconds

**3. Assisted GPS (A-GPS) - SET-Assisted**
- SUPL provides assistance data
- Device sends measurements to network
- Network calculates position
- Better accuracy
- TTFF: 5-10 seconds

**4. Network Positioning**
- Cell tower triangulation
- WiFi fingerprinting
- No GPS required
- Accuracy: 100-1000m
- TTFF: 1-2 seconds

---

# PART 2: DEEP DIVE

---

## Slide 11: Complete Location Request Flow

**High Accuracy Location Request - End-to-End**

```
═══════════════════════════════════════════════════════════
PHASE 1: APPLICATION REQUEST
═══════════════════════════════════════════════════════════

1. User opens Google Maps app
2. App calls LocationManager.requestLocationUpdates()
   - Priority: PRIORITY_HIGH_ACCURACY
   - Interval: 10 seconds

═══════════════════════════════════════════════════════════
PHASE 2: ANDROID FRAMEWORK PROCESSING
═══════════════════════════════════════════════════════════

3. LocationManagerService receives request
4. Checks permissions (ACCESS_FINE_LOCATION)
5. Selects provider: GnssLocationProvider (GPS)
6. GnssLocationProvider.enable()

═══════════════════════════════════════════════════════════
PHASE 3: HAL COMMUNICATION
═══════════════════════════════════════════════════════════

7. JNI call: android_location_GnssLocationProvider.cpp
8. GNSS HAL: IGnss.setCallback()
9. GNSS HAL: IGnss.setPositionMode()
   - Mode: MS_BASED (SUPL-assisted)
   - Recurrence: PERIODIC
   - Interval: 10000ms
10. GNSS HAL: IGnss.start()

═══════════════════════════════════════════════════════════
PHASE 4: QUALCOMM VENDOR LAYER
═══════════════════════════════════════════════════════════

11. Qualcomm GNSS HAL receives start request
12. Location Integration API: startTracking()
13. IPC to GNSS Daemon (gnss@2.1-service)
14. Location API: startTracking() call

═══════════════════════════════════════════════════════════
PHASE 5: SUPL SESSION (Parallel to GNSS)
═══════════════════════════════════════════════════════════

15. GNSS Engine triggers SUPL client
16. SUPL Client → SUPL Server: SUPL START
17. SLP → SET: SUPL RESPONSE
18. SET → SLP: SUPL POS INIT (request assistance)
19. SLP → SET: SUPL POS (Assistance Data)
    - Satellite ephemeris (precise orbits)
    - Satellite almanac (all satellites)
    - Reference time (GPS time)
    - Reference location (approximate)
    - Ionospheric model
    - Acquisition assistance (Doppler, code phase)

═══════════════════════════════════════════════════════════
PHASE 6: GNSS SIGNAL ACQUISITION
═══════════════════════════════════════════════════════════

20. GNSS Engine receives assistance data
21. Measurement Engine uses assistance to:
    - Know which satellites are visible
    - Know expected Doppler frequency
    - Know expected code phase
22. RF Frontend: Receive satellite signals
23. Acquisition Engine: Search for satellites
    - Uses acquisition assistance
    - Finds satellites in 1-2 seconds (vs 30+ sec standalone)
24. Tracking Channels: Lock onto 4+ satellites
25. Correlators: Track code and carrier phase

═══════════════════════════════════════════════════════════
PHASE 7: MEASUREMENT GENERATION
═══════════════════════════════════════════════════════════

26. Measurement Engine generates for each satellite:
    - Pseudorange: Time delay × speed of light
    - Doppler shift: Satellite velocity measurement
    - Carrier phase: High-precision phase measurement
    - C/N0: Signal strength (35-45 dB-Hz typical)
    - Measurement timestamp

═══════════════════════════════════════════════════════════
PHASE 8: POSITION CALCULATION (SET-Based Mode)
═══════════════════════════════════════════════════════════

27. Position Engine receives measurements
28. Applies corrections:
    - Satellite clock error (from ephemeris)
    - Ionospheric delay (from ionospheric model)
    - Tropospheric delay (from mathematical model)
    - Relativistic effects
29. Trilateration algorithm:
    - Least squares solution
    - Solve for X, Y, Z, Clock Bias
    - Iterative refinement
30. Kalman Filter:
    - State prediction
    - Measurement update
    - Smoothing
31. Output:
    - Latitude: 37.422408°
    - Longitude: -122.084068°
    - Altitude: 32m
    - Accuracy: 4.2m (HDOP)
    - Speed: 15 m/s
    - Bearing: 045°
    - Timestamp: GPS time

═══════════════════════════════════════════════════════════
PHASE 9: POSITION DELIVERY TO FRAMEWORK
═══════════════════════════════════════════════════════════

32. Location API → Location HAL callback
33. IGnssCallback.gnssLocationCb()
34. JNI → GnssLocationProvider
35. GnssLocationProvider.handleReportLocation()
36. LocationManagerService.handleLocationChanged()

═══════════════════════════════════════════════════════════
PHASE 10: DELIVERY TO APPLICATION
═══════════════════════════════════════════════════════════

37. LocationManagerService → App via Binder callback
38. App receives Location object:
    location.getLatitude()   // 37.422408
    location.getLongitude()  // -122.084068
    location.getAccuracy()   // 4.2 meters
39. App updates map display
40. User sees current location on map

Total Time: 5-10 seconds (first fix)
Subsequent fixes: 1-2 seconds (hot start)
```

---

## Slide 12: Android Framework Layer

**LocationManagerService Deep Dive**

**Source Location:**
`frameworks/base/services/core/java/com/android/server/location/LocationManagerService.java`

**Key Responsibilities:**

**1. Provider Management**
```java
// Provider registration
void addProviderLocked(LocationProvider provider)

// Active providers
- GnssLocationProvider (GPS)
- PassiveProvider
- NetworkLocationProvider (via GMS)
```

**2. Client Management**
```java
// Register location updates
void requestLocationUpdates(LocationRequest request,
                           ILocationListener listener,
                           PendingIntent intent)

// Active client tracking
HashMap<IBinder, Receiver> mReceivers
```

**3. Permission Enforcement**
```java
// Check permissions
if (!hasPermission(ACCESS_FINE_LOCATION)) {
    throw SecurityException
}

// Background location restrictions (Android 10+)
checkBackgroundLocationAccess()
```

**4. Location Update Distribution**
```java
// Broadcast to all registered clients
void handleLocationChanged(Location location) {
    for (Receiver receiver : mReceivers.values()) {
        receiver.callLocationChangedLocked(location);
    }
}
```

---

## Slide 13: Location Manager Service Architecture

**Provider Selection Logic**

```
LocationManagerService
│
├─ Provider Selection Algorithm
│  ├─ Priority: HIGH_ACCURACY → Select GPS
│  ├─ Priority: BALANCED → Select Fused
│  ├─ Priority: LOW_POWER → Select Network
│  └─ Priority: NO_POWER → Select Passive
│
├─ Permission Checks
│  ├─ FINE_LOCATION → GPS/Network allowed
│  ├─ COARSE_LOCATION → Network only
│  └─ BACKGROUND_LOCATION → Background tracking
│
├─ Provider State Management
│  ├─ Provider enabled/disabled
│  ├─ Provider availability
│  └─ Provider location quality
│
└─ Location Fusion (via FusedLocationProvider)
   ├─ GPS weight: 0.7 (high accuracy)
   ├─ Network weight: 0.2 (power save)
   └─ Sensor weight: 0.1 (dead reckoning)
```

**Location Quality Metrics:**
- Accuracy (meters)
- Age (milliseconds since measurement)
- Provider (GPS, Network, Fused)
- Has altitude
- Has speed/bearing

---

## Slide 14: GNSS Location Provider

**GnssLocationProvider Architecture**

**Source:** `frameworks/base/services/core/java/com/android/server/location/GnssLocationProvider.java`

**Key Components:**

**1. Main Provider Class**
```java
public class GnssLocationProvider implements LocationProvider {
    // Native interface
    private native void native_init();
    private native boolean native_start();
    private native boolean native_stop();

    // Callbacks from native
    void reportLocation(Location location);
    void reportSvStatus(GnssStatus status);
}
```

**2. Sub-Providers**
- **GnssMeasurementsProvider**: Raw GNSS measurements
- **GnssNavigationMessageProvider**: Satellite navigation messages
- **GnssStatusProvider**: Satellite status (PRN, azimuth, elevation, C/N0)
- **GnssBatchingProvider**: Batched location collection

**3. Configuration**
```java
// GNSS configuration
GnssConfiguration {
    - SUPL server address (supl.google.com)
    - SUPL mode (MSA/MSB)
    - LPP profile
    - A-GLONASS enabled
    - Emergency SUPL
    - GPS lock configuration
}
```

**4. Assistance Data Management**
```java
// Inject assistance data
void injectTime(long time, long timeReference, int uncertainty);
void injectLocation(double latitude, double longitude, float accuracy);
void injectBestLocation(Location location);

// Download XTRA/PSDS data
void downloadXtraData();
```

---

## Slide 15: RIL Architecture

**Radio Interface Layer (For Cellular Positioning)**

```
┌─────────────────────────────────────────┐
│  Telephony Framework                    │
│  - TelephonyManager                     │
│  - CellLocation, CellInfo               │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  RILJ (RIL Java)                        │
│  - RIL.java                             │
│  - Processes cellular info              │
└──────────────┬──────────────────────────┘
               ↓ HIDL/AIDL
┌─────────────────────────────────────────┐
│  Radio HAL                              │
│  - IRadio.getCellInfoList()             │
│  - IRadio.getSignalStrength()           │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  Qualcomm Vendor RIL                    │
│  - QMI NAS (Network Access Service)     │
└──────────────┬──────────────────────────┘
               ↓ QMI
┌─────────────────────────────────────────┐
│  Modem Processor                        │
│  - Cell tower measurements              │
│  - Signal strength (RSSI, RSRP, RSRQ)   │
│  - Cell ID (CID, LAC, MCC, MNC)         │
└─────────────────────────────────────────┘
```

**Cell Information for Network Positioning:**
- Cell ID (CID): Unique tower identifier
- LAC/TAC: Location/Tracking Area Code
- MCC/MNC: Country/Network code
- RSSI/RSRP: Signal strength
- Timing Advance: Distance estimate
- Serving cell + neighbor cells

This data is sent to Google Location Services → Position estimate

---

## Slide 16: Radio HAL Interface

**GNSS HAL Interfaces (AOSP)**

**Location:** `hardware/interfaces/gnss/2.1/`

**Key Interfaces:**

**1. IGnss - Main Interface**
```cpp
interface IGnss {
    // Lifecycle
    setCallback(IGnssCallback callback) generates (bool success);
    start() generates (bool success);
    stop() generates (bool success);
    cleanup();

    // Position mode configuration
    setPositionMode_1_1(GnssPositionMode mode,
                        GnssPositionRecurrence recurrence,
                        uint32_t minIntervalMs,
                        uint32_t preferredAccuracyMeters,
                        uint32_t preferredTimeMs,
                        bool lowPowerMode);

    // Assistance data injection
    injectTime(int64_t timeMs, int64_t timeReferenceMs, int32_t uncertaintyMs);
    injectLocation(double latitudeDegrees,
                   double longitudeDegrees,
                   float accuracyMeters);
    injectBestLocation(GnssLocation location);

    // Delete aiding data (reset)
    deleteAidingData(GnssAidingData aidingDataFlags);

    // Get extensions
    getExtensionGnssMeasurement() generates (IGnssMeasurement measurement);
    getExtensionGnssConfiguration() generates (IGnssConfiguration config);
    getExtensionGnssGeofencing() generates (IGnssGeofencing geofencing);
}
```

**2. IGnssCallback - Asynchronous Callbacks**
```cpp
interface IGnssCallback {
    // Location callback
    gnssLocationCb(GnssLocation location);

    // Satellite status
    gnssSvStatusCb(GnssSvStatus svStatus);

    // NMEA data
    gnssNmeaCb(int64_t timestamp, string nmea);

    // Capabilities
    gnssSetCapabilitiesCb(GnssCapabilities capabilities);

    // Status
    gnssStatusCb(GnssStatusValue status);
}
```

**3. GnssLocation Structure**
```cpp
struct GnssLocation {
    GnssLocationFlags gnssLocationFlags;
    double latitudeDegrees;
    double longitudeDegrees;
    double altitudeMeters;
    float speedMetersPerSec;
    float bearingDegrees;
    float horizontalAccuracyMeters;
    float verticalAccuracyMeters;
    float speedAccuracyMetersPerSecond;
    float bearingAccuracyDegrees;
    int64_t timestamp;  // milliseconds since January 1, 1970
}
```

---

## Slide 17: Qualcomm Vendor RIL

**Vendor HAL Implementation**

**Source Location:** `vendor/qcom/proprietary/gps/android/2.1/`

**Architecture:**

```
┌────────────────────────────────────────────┐
│  Gnss.cpp (HIDL Implementation)            │
│  - Implements IGnss interface              │
│  - Translates HIDL to Location API         │
└──────────────┬─────────────────────────────┘
               ↓
┌────────────────────────────────────────────┐
│  LocationAPI.cpp                           │
│  - Client-facing Location API              │
│  - Session management                      │
│  - Callback handling                       │
└──────────────┬─────────────────────────────┘
               ↓
┌────────────────────────────────────────────┐
│  LocationAPIClientBase.cpp                 │
│  - Base class for location clients         │
│  - IPC communication                       │
└──────────────┬─────────────────────────────┘
               ↓ LocIPC
┌────────────────────────────────────────────┐
│  gnss@2.1-service (Daemon)                 │
│  - Runs as system service                  │
│  - Manages location sessions               │
└──────────────┬─────────────────────────────┘
               ↓
┌────────────────────────────────────────────┐
│  GnssAdapter.cpp                           │
│  - Core GNSS logic                         │
│  - Interfaces with GNSS engine             │
│  - SUPL client integration                 │
└────────────────────────────────────────────┘
```

**Key Functions:**

**1. Start Tracking**
```cpp
bool Gnss::start() {
    // Configure position session
    LocationOptions options;
    options.minInterval = 1000;  // 1 Hz
    options.mode = GNSS_SUPL_MODE_MSB;  // SET-Based

    // Start tracking session
    mApi->startTracking(options);
}
```

**2. Position Mode Configuration**
```cpp
setPositionMode_1_1() {
    // Maps AOSP position mode to Qualcomm API
    switch (mode) {
        case STANDALONE:
            gnssMode = GNSS_SESSION_MODE_STANDALONE;
        case MS_BASED:
            gnssMode = GNSS_SESSION_MODE_MSB;  // SUPL SET-Based
        case MS_ASSISTED:
            gnssMode = GNSS_SESSION_MODE_MSA;  // SUPL SET-Assisted
    }
}
```

---

## Slide 18: QMI Protocol

**Qualcomm MSM Interface**

**QMI Services for Location:**

**1. QMI_LOC (Location Service)**
- Primary interface for GNSS
- Message-based protocol
- Request/Response + Indications

**Key Messages:**
```
QMI_LOC_START_REQ
├─ Session ID
├─ Fix recurrence (periodic/single)
├─ Horizontal accuracy threshold
├─ Intermediate reports
└─ Configuration parameters

QMI_LOC_POSITION_REPORT_IND
├─ Latitude, Longitude, Altitude
├─ Horizontal/Vertical uncertainty
├─ Heading, Speed
├─ Timestamp (GPS time)
├─ Technology mask (GNSS/Cell/WiFi)
└─ SV count used

QMI_LOC_GNSS_SV_INFO_IND
├─ Satellite PRN/ID
├─ Constellation (GPS/GLO/GAL/BDS)
├─ Elevation angle
├─ Azimuth angle
├─ Signal strength (C/N0)
└─ Satellite status (healthy/unhealthy)

QMI_LOC_INJECT_TIME_SYNC_DATA_REQ
├─ Reference counter
├─ Sensor receive time
├─ Sensor transmit time
└─ Uncertainty

QMI_LOC_INJECT_POSITION_REQ
├─ Latitude degrees
├─ Longitude degrees
├─ Horizontal circular uncertainty
└─ Timestamp

QMI_LOC_INJECT_SUPL_CERTIFICATE_REQ
├─ Certificate ID
├─ Certificate data
└─ Certificate length
```

**2. IPC Communication**
```
Application Processor ←→ Modem Subsystem
        ↕
   IPC Router / SMD
        ↕
   QMI Framework
        ↕
QMI_LOC Service (Modem)
```

---

## Slide 19: Modem Subsystem

**Qualcomm Modem Architecture**

```
┌────────────────────────────────────────────────┐
│  Modem Processor (Separate from AP)           │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  Real-Time OS (QuRT)                     │ │
│  │  - Real-time scheduling                  │ │
│  │  - Low latency                           │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  AMSS (Advanced Mobile Subscriber SW)    │ │
│  │  - 3GPP Protocol Stack (LTE/5G)          │ │
│  │  - 3GPP2 Protocol Stack (CDMA)           │ │
│  │  - Network registration                  │ │
│  │  - Cell measurement                      │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  QMI Services                            │ │
│  │  - QMI_DMS: Device Management            │ │
│  │  - QMI_NAS: Network Access               │ │
│  │  - QMI_WDS: Wireless Data                │ │
│  │  - QMI_LOC: Location                     │ │
│  │  - QMI_VOICE: Voice calls                │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │  GNSS Engine                             │ │
│  │  - Integrated with modem processor       │ │
│  │  - Shares hardware resources             │ │
│  │  - Cell-ID positioning integration       │ │
│  └──────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
```

**Benefits of Integration:**
- **Cell-ID positioning**: Modem provides cell tower info to GNSS engine
- **Time synchronization**: LTE network time helps GNSS
- **A-GNSS via Control Plane**: Alternative to SUPL
- **Power optimization**: Shared resources, coordinated duty cycling

---

## Slide 20: GNSS Engine Architecture

**Position Engine (PE) & Measurement Engine (ME)**

```
┌──────────────────────────────────────────────────────────┐
│  GNSS Software Stack (Qualcomm)                          │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Position Engine (PE)                              │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Navigation Solution                         │ │ │
│  │  │  - Least Squares Algorithm                   │ │ │
│  │  │  - Weighted Least Squares                    │ │ │
│  │  │  - Kalman Filter (Extended/Unscented)        │ │ │
│  │  │  - Solve for X, Y, Z, Clock Bias             │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Error Modeling & Corrections                │ │ │
│  │  │  - Ionospheric delay correction              │ │ │
│  │  │  - Tropospheric delay correction             │ │ │
│  │  │  - Satellite clock error                     │ │ │
│  │  │  - Relativistic effects                      │ │ │
│  │  │  - Multipath mitigation                      │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Sensor Fusion (Dead Reckoning)              │ │ │
│  │  │  - IMU integration (accelerometer, gyro)     │ │ │
│  │  │  - Magnetometer (heading)                    │ │ │
│  │  │  - Barometer (altitude)                      │ │ │
│  │  │  - Wheel tick (automotive)                   │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Measurement Engine (ME)                           │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Signal Tracking                             │ │ │
│  │  │  - DLL: Code phase tracking                  │ │ │
│  │  │  - PLL: Carrier phase tracking               │ │ │
│  │  │  - FLL: Carrier frequency tracking           │ │ │
│  │  │  - C/N0 estimation                           │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Measurement Generation                      │ │ │
│  │  │  - Pseudorange = (Rx_time - Tx_time) × c     │ │ │
│  │  │  - Pseudorange rate (Doppler)                │ │ │
│  │  │  - Carrier phase (cycles)                    │ │ │
│  │  │  - Accumulated delta range                   │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Acquisition Assistance                      │ │ │
│  │  │  - Uses SUPL assistance data                 │ │ │
│  │  │  - Expected Doppler from ephemeris           │ │ │
│  │  │  - Expected code phase from ref location     │ │ │
│  │  │  - Reduces search space dramatically         │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  SUPL Client                                       │ │
│  │  - Session management (SUPL INIT/START/POS/END)   │ │
│  │  - TLS connection to SLP                          │ │
│  │  - Assistance data parsing (RRLP/RRC/LPP)         │ │
│  │  - Measurement reporting (SET-Assisted mode)      │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## Slide 21: SUPL Session Initiation

**SUPL START - SET-Initiated Positioning**

**Trigger:** User opens navigation app requiring location

**Message Flow:**

```
1. Application → LocationManager: requestLocationUpdates()
2. GNSS Engine starts
3. GNSS Engine determines assistance needed

┌────────────────────────────────────────────────────────┐
│  SUPL START Message (SET → SLP)                        │
├────────────────────────────────────────────────────────┤
│  Message Type: SUPL START                              │
│  Session ID: 12345                                     │
│  SET Capabilities:                                     │
│    - posTechnology: agpsSETBased, agpsSETAssisted      │
│    - prefMethod: agpsSETBasedPreferred                 │
│    - posProtocol: rrlp, rrc, lpp                       │
│  Location ID:                                          │
│    - Cell ID: MCC=310, MNC=410, LAC=1234, CID=56789    │
│    - Status: current                                   │
│  QoP (Quality of Position):                            │
│    - horizontalAccuracy: 10 meters                     │
│    - verticalAccuracy: 30 meters                       │
│    - maxLocAge: 0 (fresh)                              │
│    - responseTime: 10 seconds                          │
└────────────────────────────────────────────────────────┘
         ↓ (over TLS connection)
┌────────────────────────────────────────────────────────┐
│  SUPL RESPONSE (SLP → SET)                             │
├────────────────────────────────────────────────────────┤
│  Message Type: SUPL RESPONSE                           │
│  Session ID: 12345                                     │
│  posMethod: agpsSETBased (confirmed)                   │
└────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────┐
│  SUPL POS INIT (SET → SLP)                             │
├────────────────────────────────────────────────────────┤
│  Message Type: SUPL POS INIT                           │
│  Session ID: 12345                                     │
│  SET Capabilities: (detailed)                          │
│    - GPS L1 C/A supported                              │
│    - GLONASS L1 supported                              │
│    - Galileo E1 supported                              │
│    - BeiDou B1 supported                               │
│  Requested Assistance Data:                            │
│    - almanac: true                                     │
│    - utcModel: true                                    │
│    - ionosphericModel: true                            │
│    - navigationModel: true (ephemeris)                 │
│    - referenceTime: true                               │
│    - referenceLocation: true                           │
│    - acquisitionAssistance: true                       │
│  Location ID: (current cell)                           │
└────────────────────────────────────────────────────────┘

Session established → Ready for positioning
```

---

## Slide 22: Assistance Data Delivery

**SUPL POS - Assistance Data Transfer**

```
┌────────────────────────────────────────────────────────┐
│  SUPL POS (SLP → SET) - Assistance Data                │
├────────────────────────────────────────────────────────┤
│  Message Type: SUPL POS                                │
│  Session ID: 12345                                     │
│                                                        │
│  Payload: LPP Message (for LTE)                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │  LPP: Provide Assistance Data                    │ │
│  │                                                  │ │
│  │  1. REFERENCE TIME                               │ │
│  │     - GPS Week: 2245                             │ │
│  │     - GPS Time of Week: 345678.500 seconds       │ │
│  │     - Uncertainty: ±50 nanoseconds               │ │
│  │     - Network time: UTC synchronized             │ │
│  │                                                  │ │
│  │  2. REFERENCE LOCATION                           │ │
│  │     - Latitude: 37.4220° N                       │ │
│  │     - Longitude: 122.0841° W                     │ │
│  │     - Uncertainty: ±1000 meters (semi-major)     │ │
│  │     - Confidence: 68%                            │ │
│  │                                                  │ │
│  │  3. IONOSPHERIC MODEL (Klobuchar)                │ │
│  │     - α0, α1, α2, α3: [parameters]               │ │
│  │     - β0, β1, β2, β3: [parameters]               │ │
│  │     Purpose: Correct ionospheric delay           │ │
│  │                                                  │ │
│  │  4. UTC MODEL                                    │ │
│  │     - A0: UTC offset parameter                   │ │
│  │     - A1: UTC rate parameter                     │ │
│  │     - Leap seconds: 18                           │ │
│  │                                                  │ │
│  │  5. ALMANAC DATA (All satellites)                │ │
│  │     For each satellite (31 GPS satellites):      │ │
│  │     - PRN: 1-32                                  │ │
│  │     - Health: 0 (healthy)                        │ │
│  │     - Eccentricity (e)                           │ │
│  │     - Inclination offset (δi)                    │ │
│  │     - Rate of right ascension (Ω̇)                │ │
│  │     - Semi-major axis (√A)                       │ │
│  │     - Longitude of ascending node (Ω0)           │ │
│  │     - Argument of perigee (ω)                    │ │
│  │     - Mean anomaly (M0)                          │ │
│  │     - af0, af1: Clock correction parameters      │ │
│  │     Valid: 6 months                              │ │
│  │                                                  │ │
│  │  6. EPHEMERIS DATA (Visible satellites)          │ │
│  │     For each visible satellite (e.g., PRN 5):    │ │
│  │     - PRN: 5                                     │ │
│  │     - URA: 2.4 meters (user range accuracy)      │ │
│  │     - Health: 0x00 (healthy)                     │ │
│  │     - IODE: Issue of Data Ephemeris              │ │
│  │     - Precise orbital parameters:                │ │
│  │       * Toe: Time of ephemeris                   │ │
│  │       * √A: Semi-major axis                      │ │
│  │       * e: Eccentricity                          │ │
│  │       * i0: Inclination angle                    │ │
│  │       * Ω0: Longitude of ascending node          │ │
│  │       * ω: Argument of perigee                   │ │
│  │       * M0: Mean anomaly at reference time       │ │
│  │       * Δn: Mean motion difference               │ │
│  │       * IDOT: Rate of inclination                │ │
│  │       * Ω̇: Rate of right ascension               │ │
│  │       * Cuc, Cus, Crc, Crs, Cic, Cis: Corrections│ │
│  │     - Clock correction parameters:               │ │
│  │       * af0, af1, af2: Polynomial coefficients   │ │
│  │       * Toc: Clock reference time                │ │
│  │       * TGD: Group delay                         │ │
│  │     Valid: 4 hours                               │ │
│  │                                                  │ │
│  │     (Repeat for PRN 7, 9, 13, 15, 18, 21, 24... )│ │
│  │     Total: 8-12 visible satellites              │ │
│  │                                                  │ │
│  │  7. ACQUISITION ASSISTANCE                       │ │
│  │     For each satellite (e.g., PRN 5):            │ │
│  │     - PRN: 5                                     │ │
│  │     - Doppler (0th order): -2534 Hz              │ │
│  │     - Doppler (1st order): 0.5 Hz/sec            │ │
│  │     - Doppler uncertainty: ±200 Hz               │ │
│  │     - Code phase: 512 chips                      │ │
│  │     - Code phase search window: 256 chips        │ │
│  │     - Azimuth: 125° (SE direction)               │ │
│  │     - Elevation: 45° (mid-sky)                   │ │
│  │                                                  │ │
│  │     Purpose: Tell receiver exactly where/how to  │ │
│  │     search for each satellite signal             │ │
│  │                                                  │ │
│  │  8. MULTI-GNSS SUPPORT (SUPL 2.0)                │ │
│  │     - GPS ephemeris: 8 satellites                │ │
│  │     - GLONASS ephemeris: 6 satellites            │ │
│  │     - Galileo ephemeris: 4 satellites            │ │
│  │     - BeiDou ephemeris: 4 satellites             │ │
│  │                                                  │ │
│  │     Total: 22 satellites with assistance         │ │
│  │                                                  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  Total Message Size: ~5-15 KB                          │
│  Transmission Time: <1 second over 4G/5G               │
└────────────────────────────────────────────────────────┘

Result: GNSS receiver has everything needed to quickly
        acquire satellites and calculate position
```

**What This Enables:**
- **Fast acquisition**: Know exactly where to look (Doppler + code phase)
- **Accurate position**: Precise ephemeris instead of broadcast
- **Error correction**: Ionospheric model reduces 10-15m error
- **Multi-constellation**: More satellites = better geometry = higher accuracy

---

## Slide 23: GPS Measurements

**Measurement Engine Output**

**After receiving assistance data and acquiring satellites:**

```
┌────────────────────────────────────────────────────────┐
│  GNSS Measurements (Generated by Measurement Engine)   │
└────────────────────────────────────────────────────────┘

Measurement Set @ GPS Time: 345678.750 seconds

Satellite PRN 5 (GPS):
├─ Constellation: GPS
├─ Pseudorange: 20,184,567.234 meters
│  Calculation: (Rx_time - Tx_time) × speed_of_light
│  Rx_time: GPS receiver clock reading when signal received
│  Tx_time: Transmit time from navigation message
├─ Pseudorange Uncertainty: ±2.1 meters
├─ Carrier Phase: 105,234,876.45 cycles
│  Higher precision than pseudorange
├─ Doppler Shift: -2,534.2 Hz
│  Indicates relative velocity
├─ Carrier-to-Noise Ratio (C/N0): 42.5 dB-Hz
│  Signal strength: Good (35-45 typical, >40 is strong)
├─ Locktime: 15.2 seconds
│  How long we've been tracking this satellite
├─ Multipath Indicator: LOW
└─ Satellite Position (from ephemeris):
   X: -15,345,678 meters (ECEF)
   Y: 10,234,567 meters (ECEF)
   Z: 20,123,456 meters (ECEF)

Satellite PRN 7 (GPS):
├─ Pseudorange: 23,456,789.123 meters
├─ Carrier Phase: 122,345,678.91 cycles
├─ Doppler: +1,234.5 Hz
├─ C/N0: 39.8 dB-Hz
└─ Satellite Position: (X, Y, Z)

Satellite PRN 13 (GPS):
├─ Pseudorange: 21,234,567.890 meters
├─ Carrier Phase: 110,987,654.32 cycles
├─ Doppler: -567.3 Hz
├─ C/N0: 41.2 dB-Hz
└─ Satellite Position: (X, Y, Z)

Satellite PRN 15 (GPS):
├─ Pseudorange: 22,345,678.901 meters
├─ Carrier Phase: 116,543,210.98 cycles
├─ Doppler: +890.1 Hz
├─ C/N0: 40.5 dB-Hz
└─ Satellite Position: (X, Y, Z)

Satellite PRN 18 (GPS):
├─ Pseudorange: 24,567,890.123 meters
├─ C/N0: 38.2 dB-Hz
└─ Satellite Position: (X, Y, Z)

Satellite PRN 3 (GLONASS):
├─ Pseudorange: 20,987,654.321 meters
├─ C/N0: 37.5 dB-Hz
└─ Satellite Position: (X, Y, Z)

Satellite PRN 12 (Galileo):
├─ Pseudorange: 22,123,456.789 meters
├─ C/N0: 43.1 dB-Hz (Galileo typically stronger)
└─ Satellite Position: (X, Y, Z)

Satellite PRN 8 (BeiDou):
├─ Pseudorange: 23,234,567.890 meters
├─ C/N0: 39.9 dB-Hz
└─ Satellite Position: (X, Y, Z)

Total Satellites Tracked: 8
├─ GPS: 5 satellites
├─ GLONASS: 1 satellite
├─ Galileo: 1 satellite
└─ BeiDou: 1 satellite

Geometry (GDOP): 1.8 (Excellent - lower is better)
├─ HDOP: 0.9 (Horizontal)
├─ VDOP: 1.5 (Vertical)
└─ PDOP: 1.8 (Position)

Measurement Quality: GOOD
All measurements ready for position calculation
```

**Next Step:**
- **SET-Based**: Use these measurements locally to calculate position
- **SET-Assisted**: Send these measurements to SLP for position calculation

---

## Slide 24: Position Calculation Methods

**SET-Assisted vs SET-Based**

### **Method 1: SET-Assisted (MS-Assisted)**

**"Device measures, Network calculates"**

```
┌────────────────────────────────────────────────────────┐
│  Step 1: SET sends measurements to SLP                 │
└────────────────────────────────────────────────────────┘

SUPL POS (SET → SLP):
├─ Message contains RRLP/LPP measurements
├─ For each satellite:
│  ├─ Satellite ID (PRN)
│  ├─ Pseudorange
│  ├─ Doppler
│  ├─ C/N0
│  └─ Measurement timestamp
└─ Measurement size: ~500 bytes (8 satellites)

┌────────────────────────────────────────────────────────┐
│  Step 2: SPC calculates position                       │
└────────────────────────────────────────────────────────┘

SPC has:
├─ Measurements from SET
├─ Precise satellite positions (ephemeris)
├─ Reference station corrections (if available)
├─ Advanced error models
└─ High-performance processing

Position Calculation:
├─ Apply satellite clock corrections
├─ Apply ionospheric corrections
├─ Apply tropospheric corrections
├─ Weighted Least Squares solution
├─ Kalman Filter smoothing
└─ Accuracy: 3-5 meters (95% confidence)

┌────────────────────────────────────────────────────────┐
│  Step 3: SLP returns position to SET                   │
└────────────────────────────────────────────────────────┘

SUPL END (SLP → SET):
├─ Latitude: 37.422408°
├─ Longitude: -122.084068°
├─ Altitude: 32.5 meters
├─ Horizontal Uncertainty: ±4.2 meters
├─ Vertical Uncertainty: ±8.5 meters
├─ Confidence: 68%
└─ Timestamp: GPS time
```

**Advantages:**
- Works with simpler GPS receivers
- Better accuracy (network has superior data/processing)
- Lower device computational cost
- Weaker signals acceptable

**Disadvantages:**
- Privacy: Network knows location
- Requires network connectivity for every fix
- Network dependency

---

### **Method 2: SET-Based (MS-Based)**

**"Device measures and calculates"**

```
┌────────────────────────────────────────────────────────┐
│  Step 1: SET receives assistance data from SLP         │
└────────────────────────────────────────────────────────┘

(Already received via SUPL POS - see Slide 22)

┌────────────────────────────────────────────────────────┐
│  Step 2: SET makes GPS measurements locally            │
└────────────────────────────────────────────────────────┘

(See Slide 23 for measurement details)

┌────────────────────────────────────────────────────────┐
│  Step 3: SET Position Engine calculates position       │
└────────────────────────────────────────────────────────┘

Position Engine (on device) executes:

A. Load satellite positions from ephemeris
   For each satellite i, at measurement time t:
   ├─ Calculate satellite ECEF position (Xi, Yi, Zi)
   ├─ Using Kepler's equations
   └─ Apply ephemeris parameters

B. Apply corrections to pseudoranges
   ρi_corrected = ρi_measured - corrections
   ├─ Satellite clock error: δtsv (from ephemeris)
   ├─ Ionospheric delay: Δiono (from Klobuchar model)
   ├─ Tropospheric delay: Δtropo (Saastamoinen model)
   └─ Relativistic effect: Δrel

C. Initial position estimate
   ├─ Use reference location from assistance data
   └─ (37.422°N, 122.084°W, 0m) as starting point

D. Iterative Least Squares (3-5 iterations)
   For iteration k:
   ├─ Compute predicted pseudorange to each satellite
   ├─ Compare to measured pseudorange (residuals)
   ├─ Form observation matrix H
   ├─ Solve: Δx = (H^T W H)^(-1) H^T W Δρ
   │  Where: Δx = [Δlat, Δlon, Δalt, Δclock_bias]
   ├─ Update position estimate
   └─ Check convergence (residuals < threshold)

E. Kalman Filter (for smoothing over time)
   ├─ State: [position, velocity, clock_bias, clock_drift]
   ├─ Prediction: x̂(k|k-1) = F·x̂(k-1) + w
   ├─ Update: x̂(k) = x̂(k|k-1) + K·[z - H·x̂(k|k-1)]
   └─ Covariance update: P(k) = (I - K·H)·P(k|k-1)

F. Calculate accuracy estimates
   ├─ Horizontal DOP from geometry
   ├─ Vertical DOP from geometry
   ├─ Uncertainty = HDOP × measurement_noise
   └─ Confidence level: 68% (1σ)

Result:
├─ Latitude: 37.422410° ±0.00004° (±4.5m)
├─ Longitude: -122.084070° ±0.00004° (±4.5m)
├─ Altitude: 31.8m ±8.2m
├─ Velocity: 15.2 m/s
├─ Bearing: 045° (NE)
└─ Calculation time: <100ms on Hexagon DSP

┌────────────────────────────────────────────────────────┐
│  Step 4: SET reports position (optional)               │
└────────────────────────────────────────────────────────┘

SUPL END (SET → SLP):
├─ Session ID: 12345
├─ Optional: Include calculated position
└─ Session termination

Position delivered to application immediately
```

**Advantages:**
- Better privacy (device calculates)
- Works offline after receiving assistance data
- Reduced network traffic
- Faster subsequent fixes (assistance valid 4 hours)

**Disadvantages:**
- Requires more capable GPS receiver
- Higher device computational requirements
- Slightly less accurate than SET-Assisted

---

## Slide 25: SET-Assisted vs SET-Based Comparison

**Trade-offs**

| Aspect | SET-Assisted (MSA) | SET-Based (MSB) |
|--------|-------------------|-----------------|
| **Where calculated** | Network (SLP) | Device (SET) |
| **Measurements sent** | Yes (~500B per fix) | No |
| **Position sent** | No | Optional |
| **Privacy** | Low (network knows) | High (device only) |
| **Network dependency** | High (every fix) | Low (assistance reusable) |
| **Receiver complexity** | Lower | Higher |
| **CPU requirements** | Lower | Higher (Kalman filter, etc.) |
| **Accuracy** | 3-5m (excellent) | 5-10m (very good) |
| **Power (per fix)** | Lower (offload) | Higher (local compute) |
| **Weak signal** | Better | Good |
| **Offline capability** | No | Yes (4 hours) |
| **Use case** | Emergency services | Navigation apps |

**Hybrid Mode:**
Many devices support both:
- Use SET-Assisted for first fix (fastest, most accurate)
- Switch to SET-Based for subsequent fixes (privacy, offline)
- Fall back to SET-Assisted in weak signal conditions

---

## Slide 26: Navigation Mathematics

**Position Calculation Deep Dive**

### **The Navigation Equation**

For each satellite *i*, the pseudorange equation:

```
ρᵢ = √[(Xᵢ - X)² + (Yᵢ - Y)² + (Zᵢ - Z)²] + c·Δt + εᵢ
```

**Variables:**
- ρᵢ = Measured pseudorange to satellite i (meters)
- (Xᵢ, Yᵢ, Zᵢ) = Satellite position in ECEF coordinates (known from ephemeris)
- (X, Y, Z) = Receiver position in ECEF (unknown - what we're solving for)
- c = Speed of light = 299,792,458 m/s
- Δt = Receiver clock bias (unknown - typically ±1ms = ±300km error!)
- εᵢ = Measurement errors (ionosphere, troposphere, multipath, noise)

### **Why 4 Satellites?**

**4 equations, 4 unknowns:**
1. X (position coordinate)
2. Y (position coordinate)
3. Z (position coordinate)
4. Δt (receiver clock bias)

**With 4+ satellites:** Overdetermined system → Least squares solution

### **Least Squares Solution**

**Linearization** (Taylor series expansion around initial guess):

```
Δρᵢ = ∂ρᵢ/∂X · ΔX + ∂ρᵢ/∂Y · ΔY + ∂ρᵢ/∂Z · ΔZ + c · ΔΔt
```

**Partial derivatives:**
```
∂ρᵢ/∂X = -(Xᵢ - X₀) / rᵢ
∂ρᵢ/∂Y = -(Yᵢ - Y₀) / rᵢ
∂ρᵢ/∂Z = -(Zᵢ - Z₀) / rᵢ

where rᵢ = √[(Xᵢ - X₀)² + (Yᵢ - Y₀)² + (Zᵢ - Z₀)²]
```

**Matrix Form:**
```
┌     ┐   ┌                    ┐ ┌    ┐
│ Δρ₁ │   │ a₁  b₁  c₁  1     │ │ ΔX │
│ Δρ₂ │ = │ a₂  b₂  c₂  1     │ │ ΔY │
│ Δρ₃ │   │ a₃  b₃  c₃  1     │ │ ΔZ │
│ Δρ₄ │   │ a₄  b₄  c₄  1     │ │cΔΔt│
│ ... │   │ ...               │ └    ┘
└     ┘   └                    ┘
  Δρ    =         H            ·  Δx

where aᵢ = -(Xᵢ - X₀)/rᵢ, bᵢ = -(Yᵢ - Y₀)/rᵢ, cᵢ = -(Zᵢ - Z₀)/rᵢ
```

**Least Squares Solution:**
```
Δx = (H^T · W · H)^(-1) · H^T · W · Δρ

where W = weight matrix (diagonal, based on C/N0)
```

**Iterative Process:**
```
1. Initial guess: x₀ (from reference location)
2. Calculate predicted pseudoranges
3. Form residuals: Δρ = ρ_measured - ρ_predicted
4. Solve for Δx
5. Update: x₁ = x₀ + Δx
6. Repeat until convergence (typically 3-5 iterations)
```

### **Kalman Filter Enhancement**

**State Vector:**
```
x = [X, Y, Z, Vx, Vy, Vz, Δt, Δḟ]^T

Position (X,Y,Z)
Velocity (Vx, Vy, Vz)
Clock bias (Δt)
Clock drift (Δḟ)
```

**Prediction Step:**
```
x̂(k|k-1) = F · x̂(k-1|k-1)
P(k|k-1) = F · P(k-1|k-1) · F^T + Q

F = state transition matrix
Q = process noise covariance
```

**Update Step:**
```
Innovation: y = z - H·x̂(k|k-1)
Kalman Gain: K = P(k|k-1)·H^T·[H·P(k|k-1)·H^T + R]^(-1)
State Update: x̂(k|k) = x̂(k|k-1) + K·y
Covariance Update: P(k|k) = [I - K·H]·P(k|k-1)

R = measurement noise covariance
```

**Benefits:**
- Smooths position estimates
- Handles sensor fusion (IMU, barometer)
- Provides velocity estimates
- Predicts during GPS outages (dead reckoning)

---

## Slide 27: Error Corrections

**Error Sources and Mitigation**

### **1. Satellite Clock Error**
- **Magnitude:** ±2 meters
- **Source:** Atomic clock drift on satellite
- **Correction:** Ephemeris contains clock parameters (af0, af1, af2)
- **Formula:** δtsv = af0 + af1·(t - toc) + af2·(t - toc)²

### **2. Ionospheric Delay**
- **Magnitude:** 5-15 meters (variable, worse at midday, low elevation)
- **Source:** Signal refraction through ionosphere (50-1000 km altitude)
- **Correction:** Klobuchar model from assistance data
- **Formula:**
  ```
  Δiono = [5·10⁻⁹ + Amp·(1 - x³)] if x < 1
  Δiono = 5·10⁻⁹ if x ≥ 1

  where x depends on α, β parameters and local time
  ```
- **Dual-frequency improvement:** L1+L5 allows direct measurement → 1m error

### **3. Tropospheric Delay**
- **Magnitude:** 2-3 meters (elevation dependent)
- **Source:** Signal delay through troposphere (0-50 km altitude)
- **Correction:** Saastamoinen model
- **Formula:**
  ```
  Δtropo = (0.002277 / sin(El)) · [P + (1255/T + 0.05)·e]

  P = atmospheric pressure (hPa)
  T = temperature (K)
  e = water vapor pressure (hPa)
  El = satellite elevation angle
  ```

### **4. Relativistic Effects**
- **Magnitude:** ~7 μs/day clock difference (satellites moving at 14,000 km/h)
- **Source:** General + Special relativity
- **Correction:** Applied to satellite clock
- **Formula:** δtrel = -2·(R·V)/c² where R=position vector, V=velocity vector

### **5. Multipath Error**
- **Magnitude:** 0-10 meters (environment dependent)
- **Source:** Signal reflections (buildings, ground)
- **Mitigation:**
  - Narrow correlator spacing
  - Carrier phase smoothing
  - Multi-path detection algorithms
  - Elevation mask (ignore low satellites)

### **6. Receiver Noise**
- **Magnitude:** 1-3 meters
- **Source:** Thermal noise, quantization
- **Mitigation:**
  - Signal averaging
  - Kalman filtering
  - High-quality oscillator

### **7. Ephemeris Errors**
- **Magnitude:** 1-5 meters (broadcast ephemeris)
- **Source:** Satellite orbit prediction errors
- **Mitigation:**
  - SUPL provides precise ephemeris (updated every 2 hours)
  - DGPS/RTK for even higher precision

### **8. Geometric Dilution of Precision (GDOP)**
- **Magnitude:** Multiplier on errors (1.5-3.0 typical, lower is better)
- **Source:** Satellite geometry
- **Formula:**
  ```
  GDOP = √(σ_x² + σ_y² + σ_z² + σ_t²) / σ_uere

  HDOP = √(σ_x² + σ_y²) / σ_uere  (Horizontal)
  VDOP = σ_z / σ_uere             (Vertical)
  PDOP = √(σ_x² + σ_y² + σ_z²) / σ_uere (Position)
  ```
- **Mitigation:**
  - Multi-constellation (more satellites → better geometry)
  - Satellite selection algorithm

### **Total Error Budget**

| Error Source | Standalone GPS | With SUPL AGPS | With DGPS |
|--------------|---------------|----------------|-----------|
| Satellite clock | ±2m | ±2m | ±0.5m |
| Ionosphere | ±5-15m | ±2-5m | ±0.5m |
| Troposphere | ±2-3m | ±2-3m | ±0.5m |
| Multipath | ±1-10m | ±1-10m | ±1-5m |
| Receiver noise | ±1-3m | ±1-3m | ±0.5m |
| Ephemeris | ±2-5m | ±1-2m | ±0.1m |
| **Total RMS** | **~15m** | **~5-8m** | **~1-2m** |
| **95% confidence** | **~30m** | **~10-15m** | **~2-4m** |

**GDOP multiplier:** 1.5-3.0× (depends on geometry)

**Final Accuracy:**
- Standalone GPS: 10-30 meters (95%)
- SUPL AGPS: 5-10 meters (95%)
- DGPS/RTK: 1-5 meters (95%) or 1-2 cm (RTK)

---

## Slide 28: Multi-GNSS Support

**Multi-Constellation Positioning**

### **Supported Constellations (SUPL 2.0 + Modern Qualcomm Chips)**

| Constellation | Country | Satellites | Frequency Bands | Coverage | Status |
|--------------|---------|------------|-----------------|----------|--------|
| **GPS** | USA | 31 operational | L1 (1575.42 MHz), L5 (1176.45 MHz) | Global | Fully operational |
| **GLONASS** | Russia | 24 operational | L1 (1602 MHz), L2 (1246 MHz) | Global | Fully operational |
| **Galileo** | EU | 28 operational | E1 (1575.42 MHz), E5a (1176.45 MHz) | Global | Fully operational |
| **BeiDou** | China | 49 operational | B1 (1561.098 MHz), B2 (1207.14 MHz) | Global | Fully operational |
| **QZSS** | Japan | 7 operational | L1, L5, L6 | Asia-Pacific | Regional |
| **NavIC** | India | 7 operational | L5, S-band | India ±1500km | Regional |

**Total:** 140+ satellites available worldwide

### **Benefits of Multi-GNSS**

**1. More Visible Satellites**
```
Single GPS:        6-10 satellites visible
GPS + GLONASS:     12-18 satellites visible
GPS + GLO + GAL:   18-25 satellites visible
All systems:       25-35 satellites visible
```

**2. Improved Geometry (Lower GDOP)**
```
GPS only:          GDOP = 2.5-4.0
GPS + GLONASS:     GDOP = 1.8-2.5
All constellations:GDOP = 1.2-1.8
```

Lower GDOP → Better accuracy (GDOP multiplies errors)

**3. Faster TTFF**
- More satellites → Faster acquisition
- Cold start: 5-8 seconds (vs 15-30 seconds GPS-only)

**4. Better Urban Canyon Performance**
- Buildings block some satellites
- More total satellites → More likely to see 4+ satellites
- Critical for dense urban environments

**5. Improved Accuracy**
```
GPS only:              5-10 meters
GPS + GLONASS:         4-8 meters
GPS + GLO + GAL + BDS: 3-5 meters
```

**6. Redundancy**
- If one constellation has issues, others continue working
- Anti-jamming (harder to jam all frequencies)

### **Qualcomm Multi-GNSS Implementation**

**Tracking Channels:**
```
Qualcomm SDM865 (example):
├─ Total channels: 48
├─ GPS: Up to 32 channels
├─ GLONASS: Up to 16 channels
├─ Galileo: Up to 16 channels
├─ BeiDou: Up to 16 channels
├─ QZSS: Shared with GPS
└─ NavIC: Shared with GPS

Simultaneous tracking: All constellations
```

**Assistance Data via SUPL 2.0:**
```
SUPL POS message contains:
├─ GPS ephemeris (8-12 satellites)
├─ GLONASS ephemeris (6-8 satellites)
├─ Galileo ephemeris (4-6 satellites)
├─ BeiDou ephemeris (4-6 satellites)
├─ Acquisition assistance for all
└─ Total: 22-32 satellites assisted
```

**Position Calculation:**
- Combines measurements from all constellations
- Time system conversions:
  - GPS time (reference)
  - GLONASS time (UTC-based, +3 hours offset)
  - Galileo System Time (GPS-synchronized)
  - BeiDou Time (33 sec offset from GPS)
- Unified Kalman filter processes all measurements

---

## Slide 29: Performance Metrics

**Real-World Performance Characteristics**

### **Time To First Fix (TTFF)**

| Scenario | Standalone GPS | A-GPS (SUPL) | Improvement |
|----------|---------------|--------------|-------------|
| **Cold Start** | 30-60 seconds | 5-10 seconds | **6-10x faster** |
| **Warm Start** | 20-30 seconds | 3-5 seconds | **6-8x faster** |
| **Hot Start** | 5-10 seconds | 1-3 seconds | **3-5x faster** |
| **Factory Test** | 2-5 minutes | 5-10 seconds | **24-60x faster** |

**Definitions:**
- **Cold Start:** No almanac, ephemeris, time, or position (device off >4 hours)
- **Warm Start:** Has almanac but stale ephemeris (device off 1-4 hours)
- **Hot Start:** Recent almanac and ephemeris (device off <1 hour)
- **Factory Test:** New device, never had GPS fix

### **Accuracy Comparison**

| Method | Horizontal Accuracy (95%) | Vertical Accuracy (95%) | Best Case |
|--------|--------------------------|------------------------|-----------|
| **Standalone GPS** | 10-30 meters | 15-40 meters | 5-10 meters |
| **A-GPS (SUPL)** | 5-10 meters | 10-20 meters | 3-5 meters |
| **A-GPS Multi-GNSS** | 3-5 meters | 5-10 meters | 2-3 meters |
| **SBAS (WAAS/EGNOS)** | 1-2 meters | 2-3 meters | 0.5-1 meter |
| **DGPS** | 1-3 meters | 2-5 meters | 0.5-1 meter |
| **RTK** | 1-2 cm | 2-5 cm | 1 cm |
| **PPP** | 5-10 cm | 10-20 cm | 3-5 cm |

### **Sensitivity**

| Parameter | Qualcomm (Typical) | Requirement |
|-----------|-------------------|-------------|
| **Tracking Sensitivity** | -160 dBm | Continue tracking |
| **Acquisition Sensitivity** | -148 dBm | Acquire new satellite |
| **Cold Start Sensitivity** | -145 dBm | Initial fix |
| **Reacquisition Time** | <1 second | After signal loss |

**Comparison:**
- Standalone GPS: -142 dBm typical
- A-GPS: Can work down to -148 dBm (acquisition assistance)

### **Power Consumption**

| Operation | Power Draw | Duration | Total Energy |
|-----------|-----------|----------|--------------|
| **GPS acquisition (standalone)** | 150-200 mW | 30-60s | 4.5-12 Ws |
| **GPS acquisition (A-GPS)** | 150-200 mW | 5-10s | 0.75-2 Ws |
| **Energy Savings** | - | - | **83-93%** |
| **Continuous tracking** | 80-120 mW | Continuous | - |
| **Batching mode** | 15-30 mW | Continuous | 75% savings |
| **Geofencing (HW)** | 1-5 mW | Continuous | 95% savings |

### **Success Rate**

| Environment | Standalone GPS | A-GPS (SUPL) | Multi-GNSS A-GPS |
|-------------|---------------|--------------|------------------|
| **Open Sky** | 95-99% | 98-100% | 99-100% |
| **Light Foliage** | 80-90% | 90-95% | 95-98% |
| **Urban Canyon** | 40-60% | 60-80% | 75-90% |
| **Indoor (near window)** | 10-30% | 30-50% | 40-60% |
| **Deep Indoor** | 0-5% | 5-15% | 10-20% |

### **Latency**

| Stage | Time |
|-------|------|
| **App request → Framework** | 10-50 ms |
| **Framework → HAL** | 5-20 ms |
| **HAL → GNSS Engine** | 5-15 ms |
| **SUPL session establishment** | 200-500 ms (first time) |
| **Assistance data download** | 300-800 ms |
| **Satellite acquisition** | 1-5 seconds |
| **Position calculation** | 50-200 ms |
| **Callback to app** | 10-50 ms |
| **Total (first fix)** | 5-10 seconds |
| **Subsequent fixes** | 1-3 seconds |

### **Data Usage**

| Operation | Data Size | Frequency |
|-----------|-----------|-----------|
| **SUPL session (SET-Based)** | 5-15 KB | Per session |
| **SUPL measurements (SET-Assisted)** | 0.5 KB | Per fix (1 Hz = 1.8 MB/hour) |
| **XTRA download** | 100-500 KB | Every 7-14 days |
| **NTP time sync** | <1 KB | Periodic |

### **Real-World Example: Google Maps Navigation**

```
User opens Google Maps and starts navigation:

T+0.0s:  App requests location (HIGH_ACCURACY)
T+0.1s:  LocationManagerService activates GPS
T+0.2s:  GNSS HAL starts
T+0.3s:  SUPL START sent to server
T+0.5s:  TLS connection established
T+0.8s:  SUPL POS INIT sent
T+1.2s:  Assistance data received (10 KB)
T+1.5s:  Satellite acquisition begins
T+3.0s:  4 satellites locked
T+5.5s:  8 satellites tracked
T+6.0s:  Position calculated: 37.422408°N, 122.084068°W, ±4.2m
T+6.1s:  Location delivered to app
T+6.2s:  Map displays user location and route

Subsequent updates: Every 1 second, <100ms latency
Total data: 15 KB (initial) + 0 KB/fix (SET-Based mode)
Power: 80-120 mW continuous, ~15 hours of navigation on 3000mAh battery
```

---

## Slide 30: Summary & Conclusion

**Complete Android-Qualcomm-SUPL Location Flow**

### **The Complete Journey: App to Satellite to Position**

```
1. APPLICATION LAYER
   └─ User opens Maps → LocationManager.requestLocationUpdates()

2. ANDROID FRAMEWORK
   └─ LocationManagerService → GnssLocationProvider.enable()

3. HAL INTERFACE
   └─ IGnss.start() → Vendor HAL implementation

4. QUALCOMM VENDOR
   └─ Location API → GNSS Daemon → Location Engine

5. SUPL ASSISTANCE (Parallel)
   └─ SUPL Client → SUPL Server → Assistance Data
      ├─ Satellite ephemeris (precise orbits)
      ├─ Reference time & location
      ├─ Ionospheric corrections
      └─ Acquisition assistance (Doppler, code phase)

6. GNSS HARDWARE
   └─ RF Frontend → Acquisition → Tracking → Measurements
      ├─ Uses assistance data to acquire fast (5-10s vs 30-60s)
      └─ Tracks 8+ satellites (GPS, GLONASS, Galileo, BeiDou)

7. POSITION CALCULATION
   └─ Measurement Engine → Position Engine
      ├─ Apply corrections (ionosphere, troposphere, clock)
      ├─ Least squares trilateration
      ├─ Kalman filter smoothing
      └─ Output: Lat/Lon/Alt ±4m accuracy

8. CALLBACK CHAIN
   └─ GNSS Engine → HAL → Framework → Application

9. USER SEES LOCATION
   └─ Blue dot on map, navigation route displayed

Total Time: 5-10 seconds (first fix), 1-3 seconds (subsequent)
Accuracy: 3-10 meters (95% confidence)
Power: 80-120mW continuous tracking
```

### **Key Takeaways**

**1. Multi-Layer Architecture**
- Clean separation: App → Framework → HAL → Vendor → Hardware
- Each layer has specific responsibilities
- HIDL/AIDL provides stable vendor interface

**2. SUPL is Critical for Performance**
- **10x faster TTFF**: 5-10s vs 30-60s standalone
- **2-3x better accuracy**: Ionospheric corrections, precise ephemeris
- **90% less power**: Fast acquisition reduces active GPS time
- **Works in weak signals**: Acquisition assistance helps indoors/urban

**3. Qualcomm Integration**
- Dedicated GNSS processor (Hexagon DSP)
- Hardware acceleration (Position Engine, Measurement Engine)
- Multi-constellation support (GPS+GLONASS+Galileo+BeiDou)
- Low-power modes (batching, geofencing)
- Integration with modem (cell-ID positioning, network time)

**4. Position Calculation**
- **SET-Based**: Device calculates (privacy, offline capability)
- **SET-Assisted**: Network calculates (better accuracy, lower power)
- Least squares + Kalman filtering
- Multi-source fusion (GNSS + IMU + barometer)

**5. Error Correction is Essential**
- Ionospheric delay: 5-15m → 2-5m with model
- Tropospheric delay: 2-3m correction
- Satellite clock: ±2m correction
- Multi-GNSS improves geometry (lower GDOP)

**6. Real-World Performance**
- TTFF: 5-10 seconds (cold start with A-GPS)
- Accuracy: 3-5 meters (multi-GNSS A-GPS)
- Sensitivity: -160 dBm tracking
- Power: 80-120mW continuous, 15-30mW batching

### **Technology Evolution**

**Current State (2024-2026):**
- SUPL 2.0 widely deployed
- Multi-GNSS standard (GPS+GLO+GAL+BDS)
- Dual-frequency (L1+L5) becoming common
- Android 14+ with enhanced privacy controls

**Future Directions:**
- **SUPL 3.0**: 5G integration, enhanced IoT support
- **PPP/RTK**: Centimeter-level accuracy in smartphones
- **5G NR positioning**: Alternative to GNSS (indoor, urban)
- **AI/ML enhancement**: Improved multipath mitigation
- **Quantum-resistant security**: Post-quantum cryptography for SUPL

### **References**

**Specifications:**
- OMA SUPL 2.0: OMA-AD-SUPL-V2_0-20120417-A
- 3GPP TS 36.355: LPP (LTE Positioning Protocol)
- Android GNSS HAL: hardware/interfaces/gnss/
- GPS ICD: IS-GPS-200 (GPS Interface Control Document)

**Source Code:**
- AOSP: https://android.googlesource.com/
  - frameworks/base/services/core/java/com/android/server/location/
  - hardware/interfaces/gnss/
- Qualcomm: vendor/qcom/proprietary/gps/

**Tools:**
- GPS Test (Android app): Satellite visualization
- GNSS Logger (Google): Raw measurements
- QXDM (Qualcomm): Professional debugging

---

## Slide 31: Q&A

**Questions?**

**Topics for deeper exploration:**
- RTK (Real-Time Kinematic) implementation
- Geofencing hardware acceleration
- Raw GNSS measurements API
- 5G NR positioning
- Indoor positioning techniques
- Privacy considerations in location services

---

## Presentation Notes

**Recommended Format:**
- **PowerPoint/Google Slides**: 50-60 slides for comprehensive coverage
- **Presentation Time**: 45-60 minutes (technical audience)
- **Target Audience**: Android platform engineers, GNSS developers, system architects

**Visual Elements:**
- Architecture diagrams (use Draw.io files from repo)
- Flow diagrams (SUPL message flows)
- Comparison tables (SET-Assisted vs SET-Based)
- Performance graphs (TTFF, accuracy, power)
- Code snippets (Java/C++ for key interfaces)
- Real-world examples (Google Maps scenario)

**Delivery Tips:**
1. Start with big picture (Slide 3), then drill down
2. Use Google Maps example throughout for context
3. Live demo if possible (GPS Test app showing satellites)
4. Emphasize SUPL benefits (10x faster, 90% less power)
5. Show actual measurements (Slide 23) to make it concrete
6. Connect each layer to previous (continuity)

**Hands-On Options:**
- Show adb commands: `adb shell dumpsys location`
- Display NMEA output in real-time
- Compare TTFF with/without airplane mode (simulates cold start)

---

**End of Presentation**
