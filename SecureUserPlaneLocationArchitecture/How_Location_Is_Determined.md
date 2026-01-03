# How Location Is Determined Through SUPL

## Overview

SUPL (Secure User Plane Location) determines device location using **Assisted GPS (AGPS)** technology. This document explains the technical process of how a position fix is calculated using SUPL.

---

## The Challenge: Standalone GPS Limitations

### Traditional GPS Problems:
- **Slow Time To First Fix (TTFF)**: 30-60+ seconds cold start
- **Weak Signals**: Difficult indoors or in urban canyons
- **High Power Consumption**: Extended satellite search
- **Almanac/Ephemeris Download**: Takes 12.5-30 minutes from satellites

**Solution:** SUPL provides assistance data to accelerate and improve GPS positioning.

---

## SUPL Location Determination: Two Methods

### Method 1: SET-Assisted (MS-Assisted)
**"Device measures, Network calculates"**

1. **SPC provides assistance data** to SET (satellite info, reference location)
2. **SET makes GPS measurements** (pseudoranges, Doppler, code phase)
3. **SET sends measurements to SPC** via SUPL POS messages
4. **SPC calculates position** using measurements + reference data
5. **SPC returns position** to SET/LCS Client

**Advantages:**
- Works with simpler GPS receivers
- Better for weak signal environments
- More accurate (network has better data)
- Lower device computational load

**Disadvantages:**
- Requires network connectivity for every fix
- Privacy concerns (network knows location)
- Network dependency

---

### Method 2: SET-Based (MS-Based)
**"Device measures and calculates"**

1. **SPC provides assistance data** to SET (ephemeris, almanac, reference time)
2. **SET makes GPS measurements** locally
3. **SET calculates position** on device using GPS receiver + assistance data
4. **SET reports position** (or uses it locally)
5. **Optional: SET sends position to SPC** via SUPL END

**Advantages:**
- Better privacy (device calculates position)
- Works offline after receiving assistance data
- Reduced network traffic
- Faster subsequent fixes

**Disadvantages:**
- Requires more capable GPS receiver
- Higher device computational requirements
- Slightly less accurate than SET-Assisted

---

## Assistance Data: The Key to AGPS

### What is Assistance Data?

Assistance data is information sent from the SPC to the SET to help GPS receiver acquire satellites faster and calculate position more accurately.

### Types of Assistance Data:

#### 1. **Reference Time**
- **What:** Accurate GPS time synchronized with GPS satellite constellation
- **Why:** GPS receivers need precise time to calculate position
- **Impact:** Eliminates need to download time from satellites (saves 30+ seconds)

#### 2. **Reference Location**
- **What:** Approximate location of the device (cell tower location, last known position)
- **Why:** Helps GPS receiver know which satellites to look for
- **Impact:** Reduces satellite search space, faster acquisition

#### 3. **Satellite Ephemeris**
- **What:** Precise orbital parameters for each GPS satellite (valid ~4 hours)
- **Why:** Required to calculate satellite positions
- **Impact:** Eliminates 30-second download from satellites
- **Size:** ~72 bytes per satellite

#### 4. **Satellite Almanac**
- **What:** Coarse orbital data for all GPS satellites (valid ~6 months)
- **Why:** Helps determine which satellites are visible
- **Impact:** Reduces satellite search time
- **Size:** ~900 bytes for full constellation

#### 5. **Ionospheric Model**
- **What:** Parameters describing ionospheric delays
- **Why:** Corrects for signal delays through ionosphere
- **Impact:** Improves accuracy by 10-20 meters

#### 6. **UTC Model**
- **What:** Conversion parameters between GPS time and UTC
- **Why:** Required for time-based applications
- **Impact:** Accurate time services

#### 7. **Satellite Health/Status**
- **What:** Which satellites are operational
- **Why:** Avoid wasting time on unhealthy satellites
- **Impact:** Faster, more reliable fixes

#### 8. **Acquisition Assistance**
- **What:** Expected Doppler, code phase, and search window for each satellite
- **Why:** Tells receiver exactly where/how to search for satellites
- **Impact:** Dramatically reduces acquisition time (seconds vs minutes)

---

## LTO (Long Term Orbit) Data

### Broadcom WWRN Innovation

**LTO = Extended ephemeris predictions**

- **Duration:** 7-14 days of satellite orbital predictions
- **Source:** Worldwide Reference Network (WWRN) - global GPS monitoring stations
- **Benefit:** Device can calculate position for up to 2 weeks without network connection
- **Use Case:** Improved experience in areas with intermittent connectivity

**How it works:**
1. SET downloads 7-14 days of predicted ephemeris data (1-2 MB)
2. Stored on device
3. GPS receiver uses LTO data instead of downloading from satellites
4. Faster TTFF, lower power consumption
5. Works completely offline

---

## The Location Determination Process

### Step-by-Step: SET-Assisted Mode

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: SUPL SESSION ESTABLISHMENT                         │
└─────────────────────────────────────────────────────────────┘

1. LCS Client → SLC: Location request (MLP SLIR)
2. SLC → SET: SUPL INIT (initiate positioning session)
3. SET → SLC: Establish TLS connection
4. SET → SLC: SUPL POS INIT (report capabilities)

┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: ASSISTANCE DATA DELIVERY                           │
└─────────────────────────────────────────────────────────────┘

5. SLC → SPC: Request assistance data (ILP)
6. SPC: Generates assistance data from reference network
   - Current ephemeris for visible satellites
   - Reference time (GPS time)
   - Reference location (approximate position)
   - Ionospheric corrections
   - Acquisition assistance (Doppler, code phase)

7. SPC → SET: SUPL POS (Assistance Data)
   Encapsulated in positioning protocol:
   - RRLP (for GSM/GPRS)
   - RRC (for UMTS)
   - TIA-801 (for CDMA)
   - LPP (for LTE)

┌─────────────────────────────────────────────────────────────┐
│ PHASE 3: GPS MEASUREMENTS (on SET)                          │
└─────────────────────────────────────────────────────────────┘

8. SET GPS receiver uses assistance data:
   - Knows which satellites to search for
   - Knows expected Doppler shift
   - Knows expected code phase
   - Has accurate time reference

9. SET acquires GPS signals (typically 4+ satellites)
   - Measures pseudorange (distance to satellite)
   - Measures Doppler shift (satellite velocity)
   - Measures carrier phase (signal phase)
   - Records measurement time

   Typical measurements for each satellite:
   - Pseudorange: ~20,000 km ± accuracy
   - Doppler: ±5 kHz
   - C/N0 (signal strength): 35-45 dB-Hz

10. SET packages measurements

┌─────────────────────────────────────────────────────────────┐
│ PHASE 4: POSITION CALCULATION (on SPC)                      │
└─────────────────────────────────────────────────────────────┘

11. SET → SPC: SUPL POS (GPS Measurements)
    Contains:
    - Pseudorange measurements for each satellite
    - Doppler measurements
    - Signal strength (C/N0)
    - Measurement timestamp

12. SPC calculates position:

    a. Validate measurements (quality check)

    b. Apply corrections:
       - Satellite clock errors
       - Ionospheric delays
       - Tropospheric delays
       - Relativistic effects

    c. Solve navigation equations:
       - Trilateration using 4+ satellites
       - Least squares solution
       - Kalman filtering for improved accuracy

    d. Calculate:
       - Latitude, Longitude, Altitude
       - Horizontal accuracy (HDOP)
       - Vertical accuracy (VDOP)
       - Speed, heading (if available)
       - Timestamp (GPS time)

13. SPC → SLC: Position result (ILP)

┌─────────────────────────────────────────────────────────────┐
│ PHASE 5: POSITION DELIVERY                                  │
└─────────────────────────────────────────────────────────────┘

14. SLC → SET: SUPL END (position, accuracy estimate)
15. SLC → LCS Client: MLP SLIA (location response)
```

---

### Step-by-Step: SET-Based Mode

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: SESSION & ASSISTANCE REQUEST                       │
└─────────────────────────────────────────────────────────────┘

1. Application → SET: Request location
2. SET → SLC: SUPL START (request assistance)
3. SLC → SET: SUPL RESPONSE (acknowledge)
4. SET → SLC: SUPL POS INIT (request specific assistance data)

┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: ASSISTANCE DATA DELIVERY                           │
└─────────────────────────────────────────────────────────────┘

5. SLC → SPC: Get assistance data (ILP)
6. SPC → SET: SUPL POS (Complete assistance data)
   - Ephemeris for all visible satellites
   - Almanac for constellation
   - Reference time
   - Ionospheric model
   - UTC parameters
   - Acquisition assistance

┌─────────────────────────────────────────────────────────────┐
│ PHASE 3: LOCAL POSITION CALCULATION (on SET)                │
└─────────────────────────────────────────────────────────────┘

7. SET GPS receiver:
   - Uses assistance data to acquire satellites
   - Makes GPS measurements (pseudorange, Doppler)
   - Uses ephemeris to calculate satellite positions
   - Performs trilateration locally

8. SET calculates position:
   - Latitude, Longitude, Altitude
   - Accuracy estimate
   - Velocity (if available)
   - Timestamp

┌─────────────────────────────────────────────────────────────┐
│ PHASE 4: SESSION TERMINATION                                │
└─────────────────────────────────────────────────────────────┘

9. SET → SLC: SUPL END (optional: includes position)
10. SET → Application: Location delivered
```

---

## Position Calculation Mathematics

### Trilateration Principle

To determine position in 3D space (latitude, longitude, altitude), we need measurements to at least **4 satellites**.

**Why 4 satellites?**
- 3 satellites: Determine 3D position (x, y, z)
- 4th satellite: Solve for receiver clock bias (time error)

### Navigation Equations

For each satellite *i*, the pseudorange equation is:

```
ρᵢ = √[(xᵢ - x)² + (yᵢ - y)² + (zᵢ - z)²] + c·Δt + εᵢ
```

Where:
- ρᵢ = Measured pseudorange to satellite i
- (xᵢ, yᵢ, zᵢ) = Satellite position (from ephemeris)
- (x, y, z) = Receiver position (unknown)
- c = Speed of light (299,792,458 m/s)
- Δt = Receiver clock bias (unknown)
- εᵢ = Measurement errors

**With 4+ satellites, we have 4+ equations and 4 unknowns:**
1. x (position)
2. y (position)
3. z (position)
4. Δt (clock bias)

**Solution:** Least squares minimization or Kalman filtering

### Error Sources and Corrections

| Error Source | Magnitude | Correction Method |
|--------------|-----------|-------------------|
| Satellite clock | ±2m | Ephemeris correction parameters |
| Ionospheric delay | 5-15m | Ionospheric model from assistance data |
| Tropospheric delay | 2-3m | Mathematical model |
| Multipath | 0-10m | Signal processing, multiple measurements |
| Receiver noise | 1-3m | Filtering, averaging |
| Ephemeris errors | 1-5m | Differential corrections (DGPS) |
| Geometric dilution | Variable | GDOP calculation, satellite selection |

**Total accuracy:**
- SUPL with AGPS: 5-10 meters (95% confidence)
- Standalone GPS: 10-30 meters (95% confidence)

---

## Positioning Protocols

SUPL encapsulates positioning protocols inside SUPL POS messages:

### 1. RRLP (Radio Resource LCS Protocol)
- **Network:** GSM/GPRS/EDGE
- **3GPP Spec:** TS 44.031
- **Use:** Assistance data delivery, measurement reporting
- **Data:** Ephemeris, almanac, acquisition assistance, measurements

### 2. RRC (Radio Resource Control)
- **Network:** UMTS/WCDMA
- **3GPP Spec:** TS 25.331
- **Use:** UTRAN positioning
- **Data:** Similar to RRLP but for UMTS

### 3. TIA-801 (IS-801)
- **Network:** CDMA2000
- **3GPP2 Spec:** TIA-801
- **Use:** CDMA AGPS
- **Data:** Position determination data

### 4. LPP (LTE Positioning Protocol)
- **Network:** LTE/4G
- **3GPP Spec:** TS 36.355
- **Use:** Enhanced positioning for LTE
- **Data:** Extended GNSS support (GPS, GLONASS, Galileo, BeiDou)

---

## SUPL 2.0 Enhancements for Positioning

### A-GANSS Support

**GANSS = Galileo and Additional Navigation Satellite Systems**

SUPL 2.0 supports multiple satellite constellations:

| System | Country | Satellites | Status |
|--------|---------|------------|--------|
| **GPS** | USA | 31 | Operational |
| **GLONASS** | Russia | 24 | Operational |
| **Galileo** | EU | 30 | Operational |
| **BeiDou** | China | 35 | Operational |
| **QZSS** | Japan | 7 | Regional |
| **NavIC** | India | 7 | Regional |

**Benefits of multi-constellation:**
- More visible satellites (better geometry)
- Improved accuracy: 3-5 meters
- Better availability in urban canyons
- Faster TTFF
- Enhanced reliability

---

## Performance Metrics

### Time To First Fix (TTFF)

| Scenario | Standalone GPS | SUPL AGPS | Improvement |
|----------|---------------|-----------|-------------|
| **Cold Start** | 30-60 seconds | 5-10 seconds | 6-10x faster |
| **Warm Start** | 20-30 seconds | 3-5 seconds | 6-8x faster |
| **Hot Start** | 5-10 seconds | 1-3 seconds | 3-5x faster |
| **With LTO** | - | <1 second | Instantaneous |

**Cold Start:** No almanac, ephemeris, or time (device off >4 hours)
**Warm Start:** Has almanac but stale ephemeris (device off 1-4 hours)
**Hot Start:** Recent almanac and ephemeris (device off <1 hour)

### Accuracy Comparison

| Method | Typical Accuracy | Ideal Conditions | Poor Conditions |
|--------|-----------------|------------------|-----------------|
| Standalone GPS | 10-30m | 5-10m | 30-100m |
| SUPL AGPS | 5-10m | 3-5m | 10-30m |
| SUPL + A-GANSS | 3-5m | 2-3m | 5-15m |
| DGPS | 1-3m | 0.5-1m | 3-10m |

### Power Consumption

| Operation | Power Draw | Duration | Total Energy |
|-----------|-----------|----------|--------------|
| GPS acquisition (standalone) | 150-200 mW | 30-60s | 4.5-12 Ws |
| GPS acquisition (AGPS) | 150-200 mW | 5-10s | 0.75-2 Ws |
| **Savings** | - | - | **83-93%** |

---

## Summary: How SUPL Determines Location

**The Complete Picture:**

1. **Network provides assistance data** via SUPL POS messages
   - Satellite orbital data (ephemeris/almanac)
   - Reference time and location
   - Acquisition assistance (where to look)
   - Error correction models

2. **Device GPS receiver acquires satellites** quickly
   - Uses assistance data to know which satellites to search
   - Locks onto signals in seconds instead of minutes
   - Makes precise measurements (pseudorange, Doppler)

3. **Position calculation** (SET-Assisted or SET-Based)
   - **SET-Assisted:** Device sends measurements, network calculates position
   - **SET-Based:** Device calculates position locally using assistance data

4. **Position delivered** with accuracy estimate
   - Latitude, Longitude, Altitude
   - Accuracy (HDOP/VDOP)
   - Optional: Speed, heading, timestamp

**Key Advantages:**
- **10x faster** than standalone GPS (TTFF: 5-10s vs 30-60s)
- **More accurate** (5-10m vs 10-30m)
- **90% less power** for position acquisition
- **Works in weak signal areas** (indoors, urban canyons)
- **Enhanced with LTO** for offline positioning

**The Magic:** By providing assistance data over the IP network, SUPL eliminates the need to download slow satellite data, enabling fast, accurate, power-efficient positioning!
