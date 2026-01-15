# Google Maps GPS Acquisition Flow

> Complete 3.3-second journey from tapping the Maps icon to seeing your blue dot

## Overview

This document traces the complete technical flow of how your phone acquires a GPS location when opening Google Maps. The example uses real-world parameters:

- **Location**: Google HQ (37.422°N, 122.084°W)
- **Device**: Snapdragon 888 phone
- **Network**: 4G LTE
- **Total Time**: 3.3 seconds to first fix

---

## Sequence Diagram

```mermaid
sequenceDiagram
    participant User
    participant Android
    participant LocationMgr as LocationManagerService
    participant HAL as GNSS HAL
    participant Engine as GNSS Engine
    participant Network
    participant SUPL as SUPL Server
    participant Sats as Satellites

    Note over User,Sats: T=0.000s - Application Launch
    User->>Android: Tap Maps icon
    Android->>LocationMgr: Request high-accuracy location
    LocationMgr->>LocationMgr: Check ACCESS_FINE_LOCATION ✓
    LocationMgr->>HAL: Select GnssLocationProvider
    
    Note over User,Sats: T=0.100s - Hardware Initialization
    HAL->>Engine: Load Qualcomm HAL (IGnss)
    Engine-->>HAL: Register callbacks
    HAL->>Engine: Configure MS_BASED, 1000ms interval
    HAL->>Engine: Inject network time (±50ms)
    HAL->>Engine: Inject WiFi position (±500m)
    
    Note over User,Sats: T=0.250s - Start GPS
    HAL->>Engine: IGnss.start()
    Engine->>Engine: Power up RF frontend
    Engine->>Engine: Initialize tracking channels
    Engine-->>HAL: SUCCESS
    
    Note over User,Sats: T=0.400s - SUPL Assistance Request
    Engine->>Network: DNS lookup supl.google.com
    Network-->>Engine: 172.217.15.174
    Engine->>SUPL: TCP connect :7275 (3-way handshake)
    SUPL-->>Engine: ACK (20ms RTT)
    Engine->>SUPL: TLS handshake
    SUPL-->>Engine: Secure connection (50ms)
    
    Note over User,Sats: T=0.600s - SUPL Protocol Exchange
    Engine->>SUPL: SUPL START (capabilities, cell ID)
    SUPL-->>Engine: SUPL RESPONSE (confirm MS_BASED)
    Engine->>SUPL: SUPL POS INIT (request assistance)
    SUPL->>SUPL: Calculate visible satellites<br/>Retrieve ephemeris<br/>Calculate Doppler & code phase
    
    Note over User,Sats: T=1.100s - Receive Assistance Data
    SUPL-->>Engine: SUPL POS (~12KB)<br/>GPS: 10 sats<br/>GLONASS: 6 sats<br/>Galileo: 4 sats<br/>BeiDou: 4 sats
    
    Note over User,Sats: T=1.400s - Satellite Acquisition
    Engine->>Engine: Parse assistance data
    Engine->>Sats: Parallel search (assisted)
    Sats-->>Engine: PRN 5 acquired (-2531 Hz, code 514)
    Sats-->>Engine: PRN 7, 13, 15, 18, 21, 24, 30
    Sats-->>Engine: GLONASS & Galileo sats
    
    Note over User,Sats: T=3.000s - Position Calculation
    Engine->>Engine: Collect 12 pseudoranges<br/>12 Doppler measurements<br/>12 C/N0 values
    Engine->>Engine: Least squares iteration (50ms)
    Engine->>Engine: Kalman filter init
    Engine->>Engine: ECEF → Lat/Lon/Alt<br/>37.422401°N, -122.084063°W<br/>±4.2m accuracy
    
    Note over User,Sats: T=3.100s - Report Position
    Engine->>HAL: QMI_LOC_POSITION_REPORT_IND
    HAL->>LocationMgr: gnssLocationCb() via JNI
    LocationMgr->>LocationMgr: handleReportLocation()
    LocationMgr->>Android: handleLocationChanged()
    
    Note over User,Sats: T=3.250s - UI Update
    Android->>User: Blue dot appears!<br/>Map centers on location
    
    Note over User,Sats: T=3.300s - Cleanup
    Engine->>SUPL: SUPL END
    SUPL-->>Engine: Close TLS connection
    
    Note over User,Sats: T=4.000s+ - Continuous Tracking
    loop Every 1 second
        Sats-->>Engine: New measurements
        Engine->>Android: Updated position (no SUPL needed)
        Android->>User: Update blue dot
    end
```

---

## Detailed Timeline

### Phase 1: Application Launch (T=0.000s to T=0.250s)

| Time | Event | Details |
|------|-------|---------|
| T=0.000s | User taps Maps icon | Android launches application |
| T=0.050s | LocationManagerService | Permission check: `ACCESS_FINE_LOCATION` ✓<br/>Provider selection: GnssLocationProvider |
| T=0.100s | GNSS HAL initialization | JNI call to native code<br/>Load Qualcomm HAL (IGnss interface)<br/>Register callbacks |
| T=0.150s | Configure position mode | Mode: MS_BASED<br/>Recurrence: PERIODIC<br/>Interval: 1000 ms |
| T=0.200s | Inject assistance | Time: Network time ±50ms<br/>Location: Last WiFi position ±500m |
| T=0.250s | Start GPS | `IGnss.start()` called<br/>`QMI_LOC_START_REQ` sent to GNSS engine |

### Phase 2: SUPL Assistance (T=0.300s to T=1.100s)

| Time | Event | Details |
|------|-------|---------|
| T=0.300s | GNSS engine powers up | RF frontend ON<br/>Tracking channels initialized<br/>Response: SUCCESS |
| T=0.400s | DNS lookup | Resolve `supl.google.com` → 172.217.15.174 |
| T=0.450s | TCP connection | Connect to port 7275<br/>3-way handshake (20ms RTT) |
| T=0.500s | TLS handshake | Certificate exchange<br/>Secure connection established (50ms) |
| T=0.600s | SUPL START | Send capabilities, cell ID<br/>Message: ~500 bytes |
| T=0.650s | SUPL RESPONSE | SLP acknowledges, confirms MS_BASED mode |
| T=0.700s | SUPL POS INIT | Request ephemeris, almanac, acquisition assistance |
| T=0.800s | Server generates assistance | Calculate visible satellites<br/>Retrieve ephemeris from reference network<br/>Calculate Doppler and code phase |
| T=1.100s | SUPL POS delivery | GPS: 10 satellites<br/>GLONASS: 6 satellites<br/>Galileo: 4 satellites<br/>BeiDou: 4 satellites<br/>Total: ~12 KB downloaded |

### Phase 3: Satellite Acquisition (T=1.400s to T=3.100s)

| Time | Event | Details |
|------|-------|---------|
| T=1.400s | GNSS engine receives assistance | Parse and store data<br/>Load acquisition parameters |
| T=1.450s | Satellite acquisition begins | Parallel search using assistance data |
| T=1.500s | First satellite acquired (PRN 5) | Expected: -2534 Hz, code 512<br/>Found: -2531 Hz, code 514 ✓ |
| T=1.600s - T=2.500s | More satellites acquired | PRN 7, 13, 15, 18, 21, 24, 30<br/>Plus GLONASS and Galileo satellites<br/>Total: 12 satellites tracked |
| T=3.000s | First measurement epoch | 12 pseudoranges<br/>12 Doppler measurements<br/>12 carrier phases<br/>12 C/N0 values |
| T=3.050s | Position calculation | Load satellite positions<br/>Apply corrections<br/>Least squares (3-4 iterations)<br/>Kalman filter initialization<br/>ECEF → Lat/Lon/Alt<br/>Computation time: 50ms |
| T=3.100s | Position reported to Android | `QMI_LOC_POSITION_REPORT_IND`<br/>GNSS daemon → Location API → HAL<br/>`IGnssCallback.gnssLocationCb()`<br/>JNI → GnssLocationProvider |

**Calculated Position:**
- Latitude: 37.422401°N (error: -0.7m)
- Longitude: -122.084063°W (error: +0.5m)
- Altitude: 31.8 m
- Accuracy: ±4.2m (HDOP: 0.9)

### Phase 4: UI Update and Cleanup (T=3.150s to Done)

| Time | Event | Details |
|------|-------|---------|
| T=3.150s | Framework processes location | `GnssLocationProvider.handleReportLocation()`<br/>`LocationManagerService.handleLocationChanged()`<br/>Create Location object |
| T=3.200s | Location delivered to Maps | LocationListener callback<br/>Executed on Maps' main thread |
| T=3.250s | Maps updates UI | **Blue dot appears!**<br/>Map centers on your location<br/>"Your location" marker |
| T=3.300s | SUPL session terminates | SUPL END sent<br/>TLS connection closed |
| T=4.000s onwards | Subsequent fixes | Every 1 second: new position<br/>No SUPL needed (ephemeris valid 4 hours)<br/>Each fix: ~100ms measurement → callback |

---

## Key Components

### Software Stack
- **Android Framework**: LocationManagerService, GnssLocationProvider
- **HAL Layer**: Hardware Abstraction Layer (IGnss interface)
- **GNSS Engine**: Qualcomm GNSS firmware
- **Application**: Google Maps with LocationListener

### Network Protocol
- **SUPL (Secure User Plane Location)**: Protocol for A-GPS assistance
- **Server**: supl.google.com:7275
- **Transport**: TCP + TLS 1.2
- **Payload**: ~12 KB assistance data

### Satellite Systems
- **GPS**: 10 satellites tracked
- **GLONASS**: 6 satellites tracked
- **Galileo**: 4 satellites tracked
- **BeiDou**: 4 satellites tracked
- **Total**: 12 satellites used for position fix

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Time to First Fix (TTFF) | 3.3 seconds |
| Network Data Used | ~12 KB |
| Position Accuracy | ±4.2 meters |
| Horizontal Dilution of Precision (HDOP) | 0.9 (excellent) |
| Update Rate | 1 Hz (after first fix) |
| Ephemeris Validity | 4 hours |

---

## Notes

- **MS_BASED Mode**: Mobile Station Based - the phone calculates its own position using assistance data from the server
- **Cold Start**: This timeline assumes no valid ephemeris data cached on the device
- **Warm Start**: If ephemeris is cached, TTFF can be < 1 second (skips SUPL phases)
- **Hot Start**: If recent fix exists, TTFF can be < 0.5 seconds

---

## References

- Android GNSS HAL Documentation
- SUPL Protocol Specification (OMA-TS-ULP-V2_0_4)
- GPS Interface Control Document (IS-GPS-200)
- Qualcomm Location Suite Documentation