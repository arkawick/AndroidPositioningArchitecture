# Android Location & GNSS Architecture - Technical Presentation

This directory contains comprehensive technical documentation about Android's location and GNSS (Global Navigation Satellite System) implementation with Qualcomm hardware.

## Contents

### 1. Presentation Document
**File:** `AOSP_Location_GNSS_Architecture.md`

A complete technical presentation covering:
- Android location architecture (layers and components)
- Location providers (GPS, Network, Fused)
- GNSS HAL architecture and interfaces
- Qualcomm GNSS engine implementation
- GNSS chipset hardware architecture
- Multi-constellation support (GPS, GLONASS, Galileo, BeiDou, QZSS, NavIC)
- Location request flows and priority modes
- A-GNSS assistance systems (SUPL, XTRA, PSDS, NTP)
- Privacy and permissions model
- Geofencing implementation
- Advanced features (Raw measurements, RTK, Dead Reckoning)
- Performance metrics and debugging

### 2. Draw.io Diagrams

#### Diagram 4: Overall Location/GNSS Architecture
**File:** `4_Location_GNSS_Overall_Architecture.drawio`

Complete stack visualization showing:
- Application layer (Maps, Weather, Fitness apps)
- Location APIs (FusedLocationProvider, LocationManager, Geofencing)
- System services (LocationManagerService, providers)
- GNSS HAL interfaces (IGnss, IGnssMeasurement, etc.)
- Vendor HAL implementation
- Location Integration API (Qualcomm)
- GNSS daemon processes
- Position and Measurement engines
- GNSS chipset hardware components
- External assistance systems (SUPL, XTRA, NTP, PSDS)
- Cellular and WiFi positioning
- Satellite communication

#### Diagram 5: AOSP-Qualcomm GNSS Detailed Interaction
**File:** `5_AOSP_Qualcomm_GNSS_Interaction.drawio`

Detailed interaction diagram showing:
- Left side: Complete Android stack from apps to HAL
- Right side: Qualcomm GNSS subsystem architecture
- Position Engine components (Kalman filter, Least Squares, Sensor fusion)
- Measurement Engine (Satellite tracking, Pseudorange, Carrier phase)
- Signal processing layer (Acquisition, Tracking loops, Correlators)
- GNSS hardware (RF frontend, Baseband processor, DSP)
- Communication flows (Request and callback paths)
- Socket IPC between Android and GNSS daemon
- RF signal path to satellites

#### Diagram 6: Location Request Flows & Scenarios
**File:** `6_Location_Request_Flows.drawio`

Six comprehensive location request scenarios:

**1. High Accuracy (Navigation)**
- Uses: GPS/GNSS
- Accuracy: 3-5 meters
- Power: High (100+ mW)
- Update: 1 Hz

**2. Balanced Power (Weather App)**
- Uses: Fused (Multi-source)
- Accuracy: 10-100 meters
- Power: Medium (10-30 mW)
- Strategy: Network + occasional GPS

**3. Low Power (Background Tracking)**
- Uses: Network only (Cell/WiFi)
- Accuracy: 100-1000 meters
- Power: Very low (1-5 mW)
- No GNSS used

**4. Geofencing (Hardware-Accelerated)**
- Offloaded to GNSS chipset
- Power: 1-3 mW
- Capacity: ~100 geofences
- Event-driven wake-up

**5. Emergency Location Service (ELS)**
- Bypasses permissions
- Forces high-accuracy GNSS
- Uses all assistance data
- Target: <10 sec TTFF

**6. GNSS Raw Measurements (RTK/Research)**
- Provides raw satellite data
- Pseudorange, carrier phase
- For centimeter-level accuracy
- Research and development

Includes comparison matrix of all modes.

## Key Concepts Covered

### Location Architecture
- Multi-provider architecture
- FusedLocationProvider algorithm
- Permission-based access control
- Background location restrictions

### GNSS Technology
- Multi-constellation support
- Satellite signal acquisition and tracking
- Position calculation algorithms
- Kalman filtering and sensor fusion

### Qualcomm GNSS Implementation
- Position Engine (PE) architecture
- Measurement Engine (ME) implementation
- Hardware geofencing offload
- IPA (Integrated Packet Accelerator)

### Assistance Systems
- SUPL (Secure User Plane Location)
- XTRA (Extended Prediction Orbit Data)
- PSDS (Predicted Satellite Data Service)
- NTP time injection

### Advanced Features
- Raw GNSS measurements API
- RTK (Real-Time Kinematic) support
- Dead Reckoning with IMU fusion
- Indoor-outdoor transitions

## Performance Metrics

### Time To First Fix (TTFF)
- Hot Start: 1-5 seconds
- Warm Start: 10-30 seconds
- Cold Start: 30-60 seconds
- With A-GNSS: 3-15 seconds

### Accuracy
- Standard GPS: 3-5 meters
- Multi-GNSS: 2-3 meters
- RTK: 1-2 cm
- Network: 100-1000 meters

### Power Consumption
- High Accuracy: 100+ mW
- Balanced: 10-30 mW
- Low Power: 1-5 mW
- Geofence: 1-3 mW

## How to Use This Material

### For Presentations
1. **Introduction (5 min)**: Use Diagram 4 to show overall architecture
2. **Deep Dive (15 min)**: Use Diagram 5 to explain GNSS signal processing
3. **Practical Examples (10 min)**: Use Diagram 6 to show different use cases
4. **Q&A (10 min)**: Reference the MD document for technical details

### For Development
- Reference HAL interface definitions
- Understand location request parameters
- Learn power optimization strategies
- Debug location issues

### For System Integration
- Understand vendor HAL requirements
- Configure assistance data servers
- Optimize geofencing usage
- Implement privacy controls

## Source Code References

### AOSP Source Locations

```
frameworks/base/services/core/java/com/android/server/location/
├── LocationManagerService.java       # Main location service
├── GnssLocationProvider.java         # GNSS provider
├── GnssMeasurementsProvider.java     # Raw measurements
├── GeofenceManager.java              # Geofence management
└── NetworkLocationProvider.java      # Network location

frameworks/base/location/java/android/location/
├── LocationManager.java              # Public API
├── Location.java                     # Location object
├── LocationRequest.java              # Request parameters
├── GnssMeasurement.java              # Raw GNSS data
└── Geofence.java                     # Geofence definition

hardware/interfaces/gnss/
├── 1.0/ 1.1/ 2.0/ 2.1/              # HIDL interfaces
│   ├── IGnss.hal
│   ├── IGnssCallback.hal
│   ├── IGnssMeasurement.hal
│   ├── IGnssGeofencing.hal
│   └── IGnssConfiguration.hal
└── aidl/                             # AIDL interfaces (Android 12+)
```

### Qualcomm Vendor Locations

```
vendor/qcom/proprietary/gps/
├── android/
│   ├── 2.1/                          # HAL implementation
│   │   ├── location_api/
│   │   │   ├── LocationAPI.cpp
│   │   │   ├── GnssAdapter.cpp
│   │   │   └── LocationAPIClientBase.cpp
│   │   └── Gnss.cpp
│   └── AGnss.cpp                     # Assisted GNSS
├── core/
│   ├── LocApiBase.cpp
│   └── loc_core_log.cpp
└── utils/
    ├── LocTimer.cpp
    └── MsgTask.cpp
```

## Debugging and Tools

### ADB Commands

```bash
# Enable GNSS verbose logging
adb shell setprop persist.vendor.location.debug.level 3
adb shell setprop log.tag.LocSvc VERBOSE

# View GNSS logs
adb logcat -b main -s GnssLocationProvider:V
adb logcat -s LocSvc:V

# Location service status
adb shell dumpsys location

# Current location
adb shell dumpsys location | grep "last location"

# Satellite status
adb shell dumpsys location | grep -A 30 "GNSS Status"

# Enable/disable location
adb shell settings put secure location_mode 3  # Enable
adb shell settings put secure location_mode 0  # Disable

# Force XTRA download
adb shell am broadcast -a com.android.internal.location.ALARM_WAKEUP

# Mock location (testing)
adb shell appops set <package> android:mock_location allow
```

### Location Settings

```bash
# Check location providers
adb shell settings get secure location_providers_allowed

# Location mode values
# 0 = Off
# 1 = Device only (GPS)
# 2 = Battery saving (Network)
# 3 = High accuracy (GPS + Network)

# Background location scan throttling
adb shell settings get global location_background_throttle_interval_ms
```

### GNSS Testing Tools

**1. GPS Test (Android App)**
- Real-time satellite view
- Signal strength (C/N0)
- Accuracy visualization
- NMEA sentence viewer
- Multi-constellation display

**2. GNSS Logger (Google)**
- Raw measurement recording
- Research data collection
- Export to CSV/RINEX
- Open source

**3. GPSTest+** (Open Source)
- Dual-frequency support
- Satellite health
- Sky plot visualization
- Data export

**4. QXDM (Qualcomm Professional)**
- Low-level GNSS debugging
- Detailed measurement logs
- Performance analysis
- Field testing

## API Usage Examples

### High Accuracy Location Request

```java
LocationRequest request = LocationRequest.create()
    .setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY)
    .setInterval(1000)              // 1 second
    .setFastestInterval(500)        // 500 ms
    .setSmallestDisplacement(0);    // Every update

fusedLocationClient.requestLocationUpdates(
    request,
    locationCallback,
    Looper.getMainLooper()
);
```

### Geofencing Example

```java
Geofence geofence = new Geofence.Builder()
    .setRequestId("home")
    .setCircularRegion(37.7749, -122.4194, 100)  // lat, lon, radius
    .setExpirationDuration(Geofence.NEVER_EXPIRE)
    .setTransitionTypes(
        Geofence.GEOFENCE_TRANSITION_ENTER |
        Geofence.GEOFENCE_TRANSITION_EXIT
    )
    .build();

GeofencingRequest request = new GeofencingRequest.Builder()
    .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
    .addGeofence(geofence)
    .build();

geofencingClient.addGeofences(request, pendingIntent);
```

### Raw GNSS Measurements

```java
GnssMeasurementsEvent.Callback callback =
    new GnssMeasurementsEvent.Callback() {
    @Override
    public void onGnssMeasurementsReceived(GnssMeasurementsEvent event) {
        GnssClock clock = event.getClock();
        Collection<GnssMeasurement> measurements = event.getMeasurements();

        for (GnssMeasurement measurement : measurements) {
            int svid = measurement.getSvid();
            double cn0 = measurement.getCn0DbHz();
            double pseudorange = measurement.getPseudorangeRateMetersPerSecond();
            // Process raw data...
        }
    }
};

locationManager.registerGnssMeasurementsCallback(callback);
```

## Privacy and Security

### Permission Requirements

```xml
<!-- Coarse location (Network-based, ~100m) -->
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>

<!-- Fine location (GPS, ~3-5m) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>

<!-- Background location (Android 10+) -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
```

### Privacy Best Practices
1. Request minimum necessary permission
2. Explain why location is needed
3. Use foreground service for continuous tracking
4. Remove location updates when not needed
5. Support approximate location option (Android 12+)

### Location Permission States (Android 12+)
- **Precise**: Full GPS accuracy
- **Approximate**: ~1km accuracy
- **Foreground Only**: Only when app is visible
- **While Using App**: Foreground + brief background
- **One Time**: Single session only

## Power Optimization Strategies

### 1. Use Appropriate Priority
```java
// Navigation: HIGH_ACCURACY
// Weather: BALANCED_POWER_ACCURACY
// Background: LOW_POWER
// Just once: PRIORITY_NO_POWER (passive)
```

### 2. Batch Locations
```java
request.setMaxWaitTime(300000);  // 5 minutes
// Batches multiple locations, delivered together
```

### 3. Geofencing for Triggers
```java
// Use geofences instead of continuous tracking
// Wake app only when entering/exiting regions
```

### 4. Adaptive Intervals
```java
// Fast when moving, slow when stationary
// Combine with Activity Recognition API
```

## Common Issues and Solutions

### Issue: Slow TTFF (Time To First Fix)
**Solutions:**
- Ensure XTRA/PSDS data is current
- Check network connectivity for A-GNSS
- Verify time synchronization
- Clear almanac/ephemeris and restart

### Issue: Poor Accuracy Indoors
**Solutions:**
- Use PRIORITY_BALANCED (WiFi/Cell)
- Implement sensor fusion (IMU)
- Use indoor positioning systems
- Fallback to network location

### Issue: High Power Consumption
**Solutions:**
- Increase update interval
- Use geofencing instead of polling
- Lower priority mode
- Enable batching
- Remove updates when app backgrounded

### Issue: Location Denied in Background
**Solutions:**
- Request ACCESS_BACKGROUND_LOCATION
- Explain use case to user
- Use foreground service with notification
- Comply with Android 10+ restrictions

## Satellite Constellations

| System | Country | Satellites | Frequency | Status |
|--------|---------|------------|-----------|--------|
| GPS | USA | 31 | L1 (1575.42 MHz), L5 (1176.45 MHz) | Operational |
| GLONASS | Russia | 24 | L1 (1602 MHz) | Operational |
| Galileo | EU | 26 | E1 (1575.42 MHz), E5a (1176.45 MHz) | Operational |
| BeiDou | China | 35 | B1 (1561.098 MHz), B2 (1207.14 MHz) | Operational |
| QZSS | Japan | 4 | L1, L5, L6 | Regional (Asia-Pacific) |
| NavIC | India | 7 | L5, S-band | Regional (India) |

## References

- AOSP Location Source: https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/location/
- GNSS HAL Documentation: https://source.android.com/devices/sensors/gnss
- Qualcomm Location Suite: https://developer.qualcomm.com/software/location-suite
- GPS.gov: https://www.gps.gov/
- SUPL Specification: https://www.openmobilealliance.org/
- Android Location Best Practices: https://developer.android.com/training/location

## Version Information

- Created: January 2026
- Format: Markdown + Draw.io XML
- Target: Android 11+ (GNSS HAL 2.1)
- Covers: Qualcomm Snapdragon platforms

---

For technical questions or additional information, refer to the official AOSP documentation and Qualcomm developer resources.
