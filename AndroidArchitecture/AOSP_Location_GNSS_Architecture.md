# Android AOSP Location & GNSS Architecture
## Technical Deep Dive: Qualcomm Positioning Implementation

---

## Table of Contents
1. [Location Architecture Overview](#location-architecture-overview)
2. [Android Location Services](#android-location-services)
3. [Location Providers](#location-providers)
4. [GNSS HAL Architecture](#gnss-hal-architecture)
5. [Qualcomm GNSS Engine](#qualcomm-gnss-engine)
6. [GNSS Chipset Architecture](#gnss-chipset-architecture)
7. [Location Request Flows](#location-request-flows)
8. [Assistance Data Systems](#assistance-data-systems)
9. [Privacy & Permissions](#privacy-and-permissions)
10. [Geofencing Implementation](#geofencing-implementation)

---

## Location Architecture Overview

### Complete Location Stack

```
┌─────────────────────────────────────────────┐
│  Applications                               │
│  (Maps, Navigation, Weather, etc.)          │
├─────────────────────────────────────────────┤
│  Location APIs                              │
│  - FusedLocationProvider                    │
│  - LocationManager                          │
├─────────────────────────────────────────────┤
│  Location Service (system_server)           │
│  - LocationManagerService                   │
│  - GnssLocationProvider                     │
├─────────────────────────────────────────────┤
│  GNSS HAL (HIDL/AIDL)                       │
│  - IGnss, IGnssMeasurement, etc.            │
├─────────────────────────────────────────────┤
│  Vendor GNSS HAL Implementation             │
│  - Qualcomm Location HAL                    │
├─────────────────────────────────────────────┤
│  Location API (Qualcomm)                    │
│  - Location Integration API                 │
├─────────────────────────────────────────────┤
│  GNSS Engine (Qualcomm)                     │
│  - Position Engine, Measurement Engine      │
├─────────────────────────────────────────────┤
│  GNSS Chipset Hardware                      │
│  - RF Frontend, Baseband Processor          │
└─────────────────────────────────────────────┘
```

---

## Android Location Services

### LocationManagerService Architecture

**Key Components:**

1. **LocationManagerService**
   - Central system service for location
   - Manages all location providers
   - Handles client requests
   - Enforces permissions

2. **Location Providers**
   - GPS Provider (GNSS)
   - Network Provider (Cell/WiFi)
   - Passive Provider
   - Fused Provider (Google Play Services)

3. **GnssLocationProvider**
   - Native GNSS implementation
   - Interfaces with GNSS HAL
   - Manages satellite data
   - Handles GNSS measurements

### Location Request Parameters

```java
LocationRequest request = LocationRequest.create()
    .setInterval(10000)           // 10 seconds
    .setFastestInterval(5000)     // 5 seconds
    .setPriority(PRIORITY_HIGH_ACCURACY)
    .setSmallestDisplacement(10); // 10 meters
```

**Priority Modes:**
- **PRIORITY_HIGH_ACCURACY**: GPS/GNSS
- **PRIORITY_BALANCED_POWER_ACCURACY**: WiFi/Cell + GPS
- **PRIORITY_LOW_POWER**: WiFi/Cell only
- **PRIORITY_NO_POWER**: Passive listening

---

## Location Providers

### 1. GPS/GNSS Provider

```
┌────────────────────────────────────┐
│  GnssLocationProvider              │
│  ┌──────────────────────────────┐ │
│  │  GnssNative (JNI)            │ │
│  └──────────────────────────────┘ │
│  ┌──────────────────────────────┐ │
│  │  GnssMeasurementsProvider    │ │
│  └──────────────────────────────┘ │
│  ┌──────────────────────────────┐ │
│  │  GnssNavigationMessage       │ │
│  └──────────────────────────────┘ │
│  ┌──────────────────────────────┐ │
│  │  GnssStatusProvider          │ │
│  └──────────────────────────────┘ │
└────────────────────────────────────┘
```

**Features:**
- Satellite constellation support (GPS, GLONASS, Galileo, BeiDou, QZSS, NavIC)
- Raw measurements
- Navigation messages
- GNSS status callbacks
- Assistance data injection

### 2. Network Location Provider

**Data Sources:**
- Cell tower information (CID, LAC, MCC, MNC)
- WiFi access points (BSSID, RSSI)
- Bluetooth beacons
- IP-based geolocation

**Process:**
```
1. Scan WiFi/Cell towers
2. Send fingerprints to server
3. Server looks up location database
4. Return approximate location
```

### 3. Fused Location Provider

**Algorithm:**
```
Fused Location = Weight(GPS) × GPS_Location +
                 Weight(Network) × Network_Location +
                 Weight(Sensors) × Sensor_Fusion
```

**Advantages:**
- Optimal power consumption
- Smooth transitions between providers
- Intelligent provider selection
- Activity recognition integration

---

## GNSS HAL Architecture

### HAL Interfaces (Android 11+)

```
android.hardware.gnss@2.1
├── IGnss                      - Main interface
├── IGnssCallback              - Location callbacks
├── IGnssConfiguration         - Configuration
├── IGnssMeasurement           - Raw measurements
├── IGnssMeasurementCallback   - Measurement callbacks
├── IGnssNavigationMessage     - Navigation data
├── IGnssGeofencing            - Geofence management
├── IGnssDebug                 - Debug info
├── IGnssBatching              - Batched locations
├── IGnssAntennaInfo           - Antenna characteristics
└── IGnssVisibilityControl     - NFW access control
```

### IGnss Interface Methods

```cpp
interface IGnss {
    // Lifecycle
    setCallback(IGnssCallback callback);
    start();
    stop();
    cleanup();

    // Configuration
    setPositionMode(mode, recurrence, interval, ...);

    // Assistance Data
    injectTime(time, timeReference, uncertainty);
    injectLocation(latitude, longitude, accuracy);
    deleteAidingData(flags);

    // Extensions
    getExtensionGnssMeasurement();
    getExtensionGnssConfiguration();
    getExtensionGnssGeofencing();
    getExtensionGnssBatching();
}
```

### Position Modes

```cpp
enum GnssPositionMode {
    STANDALONE,      // No assistance
    MS_BASED,        // Mobile Station Based (SUPL)
    MS_ASSISTED,     // Network calculates position
};

enum GnssPositionRecurrence {
    RECURRENCE_PERIODIC,  // Continuous tracking
    RECURRENCE_SINGLE,    // Single shot
};
```

---

## Qualcomm GNSS Engine

### Location Engine Architecture

```
┌─────────────────────────────────────────┐
│  Location Integration API               │
│  (loc_api)                              │
├─────────────────────────────────────────┤
│  Location Middleware                    │
│  - Data Item Observers                  │
│  - Location Adapters                    │
│  - System Status                        │
├─────────────────────────────────────────┤
│  Location API                           │
│  - Tracking                             │
│  - Batching                             │
│  - Geofencing                           │
├─────────────────────────────────────────┤
│  IPC Layer (LocIPC)                     │
├─────────────────────────────────────────┤
│  GNSS Daemon (gnss@X.X-service)         │
├─────────────────────────────────────────┤
│  Position Engine (PE)                   │
│  - Kalman Filter                        │
│  - Least Squares                        │
│  - Velocity Estimation                  │
├─────────────────────────────────────────┤
│  Measurement Engine (ME)                │
│  - Satellite Tracking                   │
│  - Code/Carrier Phase                   │
│  - Signal Processing                    │
└─────────────────────────────────────────┘
```

### Key Components

**1. Position Engine (PE)**
- Calculates position from measurements
- Kalman filtering for smoothing
- Multi-constellation fusion
- Sensor integration (IMU, barometer)
- Dead reckoning

**2. Measurement Engine (ME)**
- Tracks satellite signals
- Measures pseudoranges
- Carrier phase tracking
- Doppler measurements
- C/N0 (Carrier-to-Noise) estimation

**3. Control Plane**
- Satellite ephemeris management
- Almanac data handling
- Time synchronization
- Assistance data processing

---

## GNSS Chipset Architecture

### Qualcomm GNSS Hardware

```
┌──────────────────────────────────────────┐
│  GNSS RF Frontend                        │
│  ┌────────────────────────────────────┐ │
│  │  L1/L5 GPS Receiver                │ │
│  │  L1 GLONASS Receiver               │ │
│  │  E1/E5a Galileo Receiver           │ │
│  │  B1/B2 BeiDou Receiver             │ │
│  └────────────────────────────────────┘ │
├──────────────────────────────────────────┤
│  GNSS Baseband Processor                 │
│  ┌────────────────────────────────────┐ │
│  │  Acquisition Engine                │ │
│  │  - Parallel search                 │ │
│  │  - Sensitivity: -160 dBm           │ │
│  └────────────────────────────────────┘ │
│  ┌────────────────────────────────────┐ │
│  │  Tracking Channels (32+)           │ │
│  │  - GPS: 24 channels                │ │
│  │  - GLONASS: 8 channels             │ │
│  │  - Galileo: 8 channels             │ │
│  │  - BeiDou: 8 channels              │ │
│  └────────────────────────────────────┘ │
│  ┌────────────────────────────────────┐ │
│  │  Correlators                       │ │
│  │  - Early/Prompt/Late               │ │
│  │  - DLL (Delay Lock Loop)           │ │
│  │  - PLL (Phase Lock Loop)           │ │
│  └────────────────────────────────────┘ │
└──────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────┐
│  GNSS Processor (Hexagon DSP)            │
│  - Signal processing                     │
│  - Position calculation                  │
│  - Navigation solution                   │
└──────────────────────────────────────────┘
```

### Multi-GNSS Constellation Support

| Constellation | Satellites | Frequency Bands | Coverage |
|--------------|------------|-----------------|----------|
| GPS (USA) | 31 | L1 (1575.42 MHz), L5 (1176.45 MHz) | Global |
| GLONASS (Russia) | 24 | L1 (1602 MHz), L2 (1246 MHz) | Global |
| Galileo (EU) | 26 | E1 (1575.42 MHz), E5a (1176.45 MHz) | Global |
| BeiDou (China) | 35 | B1 (1561.098 MHz), B2 (1207.14 MHz) | Global |
| QZSS (Japan) | 4 | L1, L5 | Asia-Pacific |
| NavIC (India) | 7 | L5, S-band | India region |

---

## Location Request Flows

### 1. High Accuracy Location Request

```
Application
    ↓
LocationManager.requestLocationUpdates()
    ↓
LocationManagerService
    ↓ (Check permissions)
PermissionManager
    ↓ (Permission granted)
LocationManagerService
    ↓ (Select provider)
GnssLocationProvider
    ↓ (JNI call)
android_location_GnssLocationProvider.cpp
    ↓
GNSS HAL (IGnss.start())
    ↓
Qualcomm GNSS HAL Implementation
    ↓
Location Integration API
    ↓
location_client_api.cpp
    ↓
IPC to GNSS Daemon
    ↓
gnss@2.1-service (Qualcomm)
    ↓
Location API
    ↓
Position Engine
    ↓
Measurement Engine
    ↓
GNSS Chipset
    ↓
Satellite Signals
```

### 2. Position Calculation Flow

```
Satellite Signals (RF)
    ↓
RF Frontend (Down-conversion)
    ↓
ADC (Analog-to-Digital Conversion)
    ↓
Acquisition Engine
    ├─ Search for satellite PRN codes
    ├─ Detect Doppler shift
    └─ Find code phase
    ↓
Tracking Loops
    ├─ DLL: Track code phase
    ├─ PLL: Track carrier phase
    └─ FLL: Track carrier frequency
    ↓
Measurement Engine
    ├─ Pseudorange = (Rx_Time - Tx_Time) × c
    ├─ Pseudorange rate (Doppler)
    ├─ Carrier phase
    └─ C/N0 (signal strength)
    ↓
Position Engine
    ├─ Collect measurements (4+ satellites)
    ├─ Least Squares algorithm
    │   - Solve for X, Y, Z, Clock Bias
    │   - Iterative refinement
    ├─ Kalman Filter
    │   - State prediction
    │   - Measurement update
    │   - State smoothing
    └─ Apply corrections
        - Ionospheric delay
        - Tropospheric delay
        - Satellite clock error
        - Relativistic effects
    ↓
Position Solution
    ├─ Latitude
    ├─ Longitude
    ├─ Altitude
    ├─ Accuracy (meters)
    ├─ Velocity
    └─ Timestamp
    ↓
Callback to Android Framework
    ↓
Application receives Location
```

---

## Assistance Data Systems

### A-GNSS (Assisted GNSS)

**Purpose:** Reduce Time To First Fix (TTFF)

### 1. SUPL (Secure User Plane Location)

```
┌─────────────────────────────────────────┐
│  SUPL Architecture                      │
│                                         │
│  Device (SUPL Enabled Terminal)         │
│      ↕                                  │
│  SUPL Location Platform (SLP)           │
│  ├─ SUPL Location Center (SLC)          │
│  └─ SUPL Positioning Center (SPC)       │
│      ↕                                  │
│  Reference Network (servers)            │
└─────────────────────────────────────────┘
```

**SUPL Assistance Data:**
- Satellite almanac (coarse orbits)
- Ephemeris data (precise orbits)
- UTC parameters
- Ionospheric model
- Reference location
- Reference time

**SUPL Session Flow:**
```
1. SUPL INIT - Server initiates or device triggers
2. SUPL POS INIT - Device sends capabilities
3. SUPL POS - Exchange assistance data
4. RRLP/LPP messages - Detailed positioning
5. SUPL END - Session termination
```

### 2. XTRA (Extended Prediction Orbit Data)

**Qualcomm XTRA System:**

```
Device → XTRA Client → Internet → XTRA Server
                                      ↓
                            GPS Ephemeris Prediction
                            (7-14 days validity)
                                      ↓
                          Download XTRA.bin file
                                      ↓
                          Inject to GNSS Engine
```

**Benefits:**
- Valid for 7-14 days
- ~100-500 KB download
- Offline capability
- Faster cold start

**XTRA Data Contents:**
- Predicted satellite positions
- Almanac data
- Satellite health status
- Time offsets
- Ionospheric parameters

### 3. NTP (Network Time Protocol)

```
GNSS Engine
    ↓ (needs accurate time)
XTRA/SNTP Client
    ↓ (request)
NTP Server (pool.ntp.org)
    ↓ (response)
Time Injection
    ↓
Reduced satellite search space
```

### 4. PSDS (Predicted Satellite Data Service)

**Google's PSDS:**
- Similar to XTRA
- Integrated with Play Services
- Automatic updates
- Better integration with Android

---

## Privacy and Permissions

### Permission Model

**Android Location Permissions:**

```java
// Coarse location (Network-based)
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>

// Fine location (GPS/GNSS)
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>

// Background location (Android 10+)
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
```

### Location Permission States (Android 11+)

1. **Precise Location** (GPS): ACCESS_FINE_LOCATION
2. **Approximate Location** (Network): ACCESS_COARSE_LOCATION
3. **Foreground Only**: Default for Android 10+
4. **Background Location**: Requires special permission
5. **One-time Permission**: Android 11+

### Emergency Location (ELS)

```
┌─────────────────────────────────────────┐
│  Emergency Location Service             │
│                                         │
│  Emergency Call (911, 112, etc.)        │
│         ↓                               │
│  ELS Triggered                          │
│         ↓                               │
│  Bypass normal permissions              │
│         ↓                               │
│  High-accuracy GNSS forced              │
│         ↓                               │
│  Location sent to PSAP                  │
│  (Public Safety Answering Point)        │
└─────────────────────────────────────────┘
```

### GNSS Visibility Control (NFW)

**Non-Framework Location (NFW) Access:**
- Modem-based location
- GNSS chipset direct access
- Emergency services
- Carrier IMS services

**Control Points:**
```
IGnssVisibilityControl HAL
    ↓
Controls who can access GNSS without framework
    ↓
User notification for NFW access
    ↓
Settings → Location → Location Services →
    "Emergency Location Service"
```

---

## Geofencing Implementation

### Geofence Architecture

```
┌─────────────────────────────────────────┐
│  Application                            │
│  - addGeofence()                        │
├─────────────────────────────────────────┤
│  LocationManager                        │
│  - GeofenceManager                      │
├─────────────────────────────────────────┤
│  Geofence Hardware HAL                  │
│  - IGnssGeofencing                      │
├─────────────────────────────────────────┤
│  Qualcomm Geofence Engine               │
│  - Hardware-based monitoring            │
│  - Low power operation                  │
└─────────────────────────────────────────┘
```

### Geofence Types

**1. Software Geofence**
- Monitored by Android framework
- Uses regular location updates
- Higher power consumption
- Flexible but less efficient

**2. Hardware Geofence**
- Monitored by GNSS chipset
- Offloaded from AP
- Low power consumption
- Limited number (~100)

### Geofence States

```
GEOFENCE_TRANSITION_ENTER    - Entering the geofence
GEOFENCE_TRANSITION_EXIT     - Exiting the geofence
GEOFENCE_TRANSITION_DWELL    - Dwelling inside
```

### Hardware Geofence Flow

```
App adds Geofence
    ↓
GeofenceManager
    ↓
IGnssGeofencing.addGeofence()
    ↓
Qualcomm Location API
    ↓
Geofence uploaded to GNSS Engine
    ↓
GNSS Engine continuously monitors
    ↓
Geofence breach detected
    ↓
IGnssGeofenceCallback.gnssGeofenceTransition()
    ↓
GeofenceManager
    ↓
PendingIntent to App
    ↓
App handles geofence event
```

---

## Advanced Features

### 1. GNSS Measurements API

**Raw GNSS Data Access:**

```java
GnssMeasurementsEvent event;
    ├─ GnssClock
    │   ├─ TimeNanos
    │   ├─ FullBiasNanos
    │   ├─ BiasNanos
    │   └─ DriftNanosPerSecond
    └─ Collection<GnssMeasurement>
        ├─ Svid (Satellite ID)
        ├─ ConstellationType
        ├─ ReceivedSvTimeNanos
        ├─ Cn0DbHz (Signal strength)
        ├─ PseudorangeRateMetersPerSecond
        ├─ AccumulatedDeltaRangeMeters
        └─ CarrierPhase
```

**Use Cases:**
- Precise Point Positioning (PPP)
- RTK (Real-Time Kinematic)
- Research and development
- GNSS signal analysis
- Indoor positioning

### 2. GNSS Navigation Messages

**Broadcast Ephemeris:**
```java
GnssNavigationMessage
    ├─ Type (L1 C/A, L5 CNAV, etc.)
    ├─ Svid
    ├─ MessageId
    ├─ SubmessageId
    └─ Data (raw navigation bits)
```

### 3. Dead Reckoning (DR)

```
┌─────────────────────────────────────────┐
│  Sensor Fusion for Dead Reckoning       │
│                                         │
│  GNSS Position                          │
│      +                                  │
│  IMU (Accelerometer + Gyroscope)        │
│      +                                  │
│  Magnetometer (Heading)                 │
│      +                                  │
│  Barometer (Altitude)                   │
│      +                                  │
│  Wheel Tick (Automotive)                │
│      ↓                                  │
│  Kalman Filter Fusion                   │
│      ↓                                  │
│  Continuous Position (GNSS outage OK)   │
└─────────────────────────────────────────┘
```

**Benefits:**
- Tunnel navigation
- Urban canyon performance
- Indoor-outdoor transitions
- Improved accuracy

### 4. RTK (Real-Time Kinematic)

**Centimeter-Level Accuracy:**

```
Base Station (Known Position)
    ↓ (Correction data)
NTRIP Caster (Internet)
    ↓
Rover (Mobile Device)
    ↓
Carrier Phase Ambiguity Resolution
    ↓
Position: ±1-2 cm accuracy
```

**RTK Data Flow:**
```
GNSS Measurements (Rover)
    +
RTCM Corrections (Base)
    ↓
Differential Processing
    ↓
Fixed Integer Ambiguities
    ↓
High Precision Position
```

---

## Performance Metrics

### Time To First Fix (TTFF)

| Mode | Typical TTFF | Description |
|------|--------------|-------------|
| Hot Start | 1-5 seconds | Valid almanac, ephemeris, position, time |
| Warm Start | 10-30 seconds | Valid almanac, approximate position |
| Cold Start | 30-60 seconds | No prior data |
| Factory Start | 2-5 minutes | No data, search all satellites |

**With Assistance (A-GNSS):**
| Mode | With XTRA | With SUPL |
|------|-----------|-----------|
| Cold Start | 5-15 sec | 3-10 sec |
| Warm Start | 3-10 sec | 2-5 sec |

### Accuracy Metrics

| Mode | Horizontal Accuracy | Vertical Accuracy |
|------|-------------------|------------------|
| Standard GPS | 3-5 meters (95%) | 5-10 meters |
| Multi-GNSS | 2-3 meters (95%) | 3-5 meters |
| SBAS (WAAS/EGNOS) | 1-2 meters | 2-3 meters |
| RTK | 1-2 cm | 2-5 cm |
| PPP | 5-10 cm | 10-20 cm |

### Sensitivity

| Parameter | Value |
|-----------|-------|
| Tracking Sensitivity | -160 dBm |
| Acquisition Sensitivity | -148 dBm |
| Cold Start Sensitivity | -145 dBm |

---

## Key Source Files

### AOSP Source Locations

```
frameworks/base/services/core/java/com/android/server/location/
├── LocationManagerService.java
├── GnssLocationProvider.java
├── GnssMeasurementsProvider.java
├── GnssNavigationMessageProvider.java
├── GnssStatusProvider.java
└── GeofenceManager.java

frameworks/base/location/java/android/location/
├── LocationManager.java
├── Location.java
├── LocationRequest.java
├── GnssMeasurement.java
├── GnssNavigationMessage.java
└── GnssStatus.java

frameworks/base/core/jni/
└── android_location_GnssLocationProvider.cpp

hardware/interfaces/gnss/
├── 1.0/
├── 1.1/
├── 2.0/
├── 2.1/
│   ├── IGnss.hal
│   ├── IGnssCallback.hal
│   ├── IGnssMeasurement.hal
│   ├── IGnssConfiguration.hal
│   └── IGnssGeofencing.hal
└── aidl/
    └── android/hardware/gnss/
```

### Qualcomm Vendor Files

```
vendor/qcom/proprietary/gps/
├── android/
│   ├── 2.1/
│   │   ├── location_api/
│   │   │   ├── LocationAPI.cpp
│   │   │   ├── LocationAPIClientBase.cpp
│   │   │   └── GnssAdapter.cpp
│   │   └── Gnss.cpp
│   └── AGnss.cpp
├── core/
│   ├── LocApiBase.cpp
│   └── loc_core_log.cpp
├── location/
│   └── location_interface.h
└── utils/
    ├── LocTimer.cpp
    └── MsgTask.cpp
```

---

## Debugging and Tools

### ADB Commands

```bash
# Enable GNSS logging
adb shell setprop persist.vendor.location.debug.level 3

# GNSS logs
adb logcat -b main -s GnssLocationProvider:V
adb logcat -b main -s LocSvc:V

# Location Manager
adb shell dumpsys location

# Current location
adb shell dumpsys location | grep "last location"

# Satellite status
adb shell dumpsys location | grep -A 20 "GNSS Status"

# Force XTRA download
adb shell am broadcast -a com.android.internal.location.ALARM_WAKEUP

# Inject test location (mock)
adb shell setprop debug.location.gps.mock true
```

### Location Settings

```bash
# Enable location
adb shell settings put secure location_mode 3

# Disable location
adb shell settings put secure location_mode 0

# Check location providers
adb shell settings get secure location_providers_allowed

# GPS logger
adb shell am start -n com.android.settings/.development.GpsLoggerActivity
```

### GNSS Tools

1. **GPS Test** (Android App)
   - Satellite view
   - Signal strength (C/N0)
   - Accuracy metrics
   - NMEA output

2. **GNSS Logger** (Google)
   - Raw measurements
   - Research tool
   - Data export

3. **QXDM** (Qualcomm)
   - Professional debugging
   - Low-level GNSS logs
   - Performance analysis

---

## Security Considerations

### Location Privacy

**Threats:**
- Location tracking
- Geolocation spoofing
- Privacy leaks

**Mitigations:**
1. **Permission System**
   - Runtime permissions
   - Background restrictions
   - One-time permissions

2. **Location Throttling**
   - Background app limits
   - Coarse location for untrusted apps

3. **GNSS Spoofing Detection**
   - Signal authentication
   - Multi-constellation consistency
   - Carrier phase validation

### Secure User Plane Location (SUPL)

**Security Features:**
- TLS encryption
- Server authentication
- Privacy mode support
- User consent

---

## Power Optimization

### Power Modes

**1. Continuous Tracking**
- Power: 50-150 mW
- Update rate: 1 Hz
- Use case: Navigation

**2. Batching Mode**
- Power: 10-30 mW
- Batch size: 100-1000 fixes
- Use case: Fitness tracking

**3. Geofence Monitoring**
- Power: 1-5 mW
- Hardware offload
- Use case: Location reminders

### Low Power Location

```
┌─────────────────────────────────────────┐
│  Low Power Strategies                   │
│                                         │
│  1. Dynamic duty cycling                │
│     - ON: 1 second                      │
│     - OFF: 9 seconds                    │
│                                         │
│  2. Adaptive positioning                │
│     - Fast movement: High rate          │
│     - Stationary: Low rate              │
│                                         │
│  3. Cell-ID positioning                 │
│     - Use cell towers when available    │
│     - GNSS only when needed             │
│                                         │
│  4. GNSS chipset optimization           │
│     - Low power tracking mode           │
│     - Reduced correlators               │
│     - Duty-cycled receiver              │
└─────────────────────────────────────────┘
```

---

## Summary

### Key Takeaways

1. **Multi-Layer Architecture**
   - Apps → Framework → HAL → Vendor → Hardware

2. **Multiple Location Sources**
   - GNSS (GPS, GLONASS, Galileo, BeiDou)
   - Network (Cell + WiFi)
   - Sensors (IMU, Barometer)

3. **Assistance Systems Critical**
   - XTRA/PSDS for fast TTFF
   - SUPL for mobile assistance
   - NTP for time synchronization

4. **Hardware Acceleration**
   - Qualcomm GNSS engine
   - Dedicated tracking channels
   - Low power modes

5. **Privacy by Design**
   - Granular permissions
   - Background restrictions
   - User transparency

---

## References

- AOSP Location Source: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/services/core/java/com/android/server/location/
- GNSS HAL: https://source.android.com/devices/sensors/gnss
- Qualcomm Location Suite: https://developer.qualcomm.com/software/location-suite
- GPS.gov: https://www.gps.gov/
- SUPL Specification: https://www.openmobilealliance.org/
