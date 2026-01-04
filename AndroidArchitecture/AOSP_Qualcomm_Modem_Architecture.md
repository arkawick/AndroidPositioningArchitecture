# Android AOSP Architecture & Qualcomm Modem Interaction
## Technical Deep Dive Presentation

---

## Table of Contents
1. [AOSP Architecture Overview](#aosp-architecture-overview)
2. [Layer-by-Layer Breakdown](#layer-by-layer-breakdown)
3. [Qualcomm Modem Architecture](#qualcomm-modem-architecture)
4. [AOSP-Modem Interaction](#aosp-modem-interaction)
5. [Radio Interface Layer (RIL)](#radio-interface-layer-ril)
6. [Hardware Abstraction Layer (HAL)](#hardware-abstraction-layer-hal)
7. [Vendor Interface (HIDL/AIDL)](#vendor-interface-hidlaidl)
8. [Call Flow Examples](#call-flow-examples)
9. [Data Flow Architecture](#data-flow-architecture)

---

## AOSP Architecture Overview

### The Android Stack (Top to Bottom)

```
┌─────────────────────────────────────────────┐
│        Applications Layer                   │
│  (System Apps, Third-Party Apps)           │
├─────────────────────────────────────────────┤
│        Application Framework                │
│  (Activity Manager, Telephony Manager,     │
│   Package Manager, Resource Manager)       │
├─────────────────────────────────────────────┤
│        Native Libraries & Runtime           │
│  (ART/Dalvik, Native Libraries, HALs)      │
├─────────────────────────────────────────────┤
│        Linux Kernel                         │
│  (Drivers, Power Management, Security)     │
├─────────────────────────────────────────────┤
│        Hardware                             │
│  (SoC, Modem, Sensors, Display)            │
└─────────────────────────────────────────────┘
```

---

## Layer-by-Layer Breakdown

### 1. Application Layer
- **Phone App** (Dialer)
- **Messaging App**
- **Settings App**
- **Third-party Apps** using telephony APIs

### 2. Application Framework Layer
Key Components:
- **TelephonyManager**: Public API for telephony services
- **PhoneInterfaceManager**: System service for phone operations
- **ActivityManager**: Manages app lifecycle
- **PackageManager**: Manages app packages

### 3. System Services Layer
- **Phone Service** (PhoneApp)
- **TelephonyRegistry**: Event broadcasting
- **ImsPhoneCallTracker**: IMS call management
- **GsmCdmaPhone**: Phone state management

### 4. RIL (Radio Interface Layer)
- **RILJ (Java)**: Java interface to RIL
- **libril.so**: Native RIL daemon
- **rild**: RIL daemon process
- **Vendor RIL**: Qualcomm-specific implementation

### 5. Hardware Abstraction Layer (HAL)
- **Radio HAL** (android.hardware.radio)
- **Data HAL** (android.hardware.radio.data)
- **Messaging HAL** (android.hardware.radio.messaging)
- **Network HAL** (android.hardware.radio.network)

### 6. Kernel Layer
- **Qualcomm IPC Router**
- **SMD (Shared Memory Driver)**
- **HSIC/PCIe drivers**
- **Network drivers**

---

## Qualcomm Modem Architecture

### Modem Subsystem Components

```
Qualcomm SoC Architecture:
┌──────────────────────────────────────┐
│  Application Processor (AP)          │
│  ┌────────────────────────────┐     │
│  │  Linux Kernel              │     │
│  │  Android Framework         │     │
│  └────────────────────────────┘     │
│              ↕ (IPC)                 │
│  ┌────────────────────────────┐     │
│  │  Modem Subsystem (MSS)     │     │
│  │  - Baseband Processor      │     │
│  │  - DSP                     │     │
│  │  - RF Transceiver          │     │
│  └────────────────────────────┘     │
└──────────────────────────────────────┘
```

### Key Modem Components

1. **Modem Processor**
   - Runs real-time OS (QuRT/REX)
   - Handles protocol stack (3G/4G/5G)
   - Manages radio resources

2. **QMI (Qualcomm MSM Interface)**
   - Message-based protocol
   - Services: DMS, NAS, WDS, UIM, VOICE, IMS
   - Transport layer for modem communication

3. **AMSS (Advanced Mobile Subscriber Software)**
   - Modem firmware
   - Protocol stack implementation
   - Radio resource management

4. **DIAG Interface**
   - Debugging and logging
   - AT command processing
   - Field testing

---

## AOSP-Modem Interaction

### Communication Path

```
Android Framework (Java)
        ↓
   RILJ (Java)
        ↓ (Socket: /dev/socket/rild)
   rild (C++)
        ↓
  libril.so (C++)
        ↓
Vendor RIL (libril-qc-hal-qmi.so)
        ↓ (QMI Messages)
  QMI Framework
        ↓ (IPC Router/SMD)
  Modem Processor
```

### IPC Mechanisms

1. **SMD (Shared Memory Driver)**
   - Legacy mechanism
   - Shared memory between AP and modem
   - Fast data transfer

2. **IPC Router**
   - Modern mechanism (Snapdragon 845+)
   - Message routing between subsystems
   - More flexible than SMD

3. **HSIC/PCIe**
   - High-speed interfaces
   - Used for data transfer
   - Lower latency

---

## Radio Interface Layer (RIL)

### RIL Architecture

```
┌─────────────────────────────────────────┐
│  Telephony Framework (Java)             │
│  - TelephonyManager                     │
│  - Phone, PhoneFactory                  │
└──────────────────┬──────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│  RILJ (RIL Java)                        │
│  - RIL.java                             │
│  - RadioIndication.java                 │
│  - RadioResponse.java                   │
└──────────────────┬──────────────────────┘
                   ↓ (HIDL/AIDL)
┌─────────────────────────────────────────┐
│  Radio HAL (C++)                        │
│  - IRadio interface                     │
│  - IRadioResponse callback              │
│  - IRadioIndication callback            │
└──────────────────┬──────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│  Vendor RIL Implementation              │
│  - Qualcomm RIL                         │
│  - QMI Client                           │
└──────────────────┬──────────────────────┘
                   ↓ (QMI)
┌─────────────────────────────────────────┐
│  Modem Processor                        │
└─────────────────────────────────────────┘
```

### RIL Request/Response Model

**Request Types:**
1. **Solicited Requests**
   - Initiated by Android framework
   - Get response callbacks
   - Example: Dial call, Send SMS

2. **Unsolicited Responses**
   - Initiated by modem
   - Async notifications
   - Example: Incoming call, Signal strength change

---

## Hardware Abstraction Layer (HAL)

### Radio HAL Modules (Android 10+)

1. **android.hardware.radio@1.6**
   - Main radio interface
   - Call control
   - Network selection

2. **android.hardware.radio.data@1.0**
   - Data connection management
   - PDP context control

3. **android.hardware.radio.messaging@1.0**
   - SMS/MMS handling
   - Cell broadcast

4. **android.hardware.radio.network@1.0**
   - Network scanning
   - Signal strength
   - Cell info

5. **android.hardware.radio.sim@1.0**
   - SIM card operations
   - UICC management

6. **android.hardware.radio.voice@1.0**
   - Voice call control
   - DTMF operations

### HAL Interface Definition Language

**HIDL (Hardware Interface Definition Language)**
- Used in Android 8.0 - 10
- C++ based
- Binder-based IPC

**AIDL (Android Interface Definition Language)**
- Used in Android 11+
- Replacing HIDL
- Better stability guarantees

---

## Vendor Interface (HIDL/AIDL)

### Treble Architecture Impact

```
┌──────────────────────────────────┐
│  Android Framework               │
│  (Updates via OTA)               │
├──────────────────────────────────┤
│  Vendor Interface (HIDL/AIDL)   │ ← Stable Interface
├──────────────────────────────────┤
│  Vendor Implementation           │
│  (Qualcomm RIL, Modem Driver)   │
│  (No update required)            │
└──────────────────────────────────┘
```

### Benefits
- **Decoupling**: Framework updates independent of vendor code
- **Stability**: Versioned interfaces
- **Security**: Process isolation
- **Modularity**: Clear boundaries

---

## Call Flow Examples

### 1. Outgoing Voice Call Flow

```
1. User dials number in Phone App
   ↓
2. Telecom Framework processes request
   ↓
3. TelephonyConnectionService invoked
   ↓
4. GsmCdmaPhone.dial() called
   ↓
5. GsmCdmaCallTracker.dial()
   ↓
6. RILJ.dial() - RIL_REQUEST_DIAL
   ↓
7. Radio HAL - IRadio.dial()
   ↓
8. Vendor RIL converts to QMI message
   ↓
9. QMI_VOICE_DIAL_CALL_REQ sent to modem
   ↓
10. Modem processes and initiates call
    ↓
11. QMI_VOICE_DIAL_CALL_RESP received
    ↓
12. Radio HAL callback - IRadioResponse.dialResponse()
    ↓
13. RILJ processes response
    ↓
14. Call state updated in framework
    ↓
15. UI updated with call in progress
```

### 2. Incoming Call Flow

```
1. Modem receives incoming call from network
   ↓
2. QMI_VOICE_ALL_CALL_STATUS_IND sent to AP
   ↓
3. Vendor RIL receives QMI indication
   ↓
4. Radio HAL - IRadioIndication.callStateChanged()
   ↓
5. RILJ.processIndication()
   ↓
6. RILJ sends RIL_REQUEST_GET_CURRENT_CALLS
   ↓
7. Radio HAL queries modem for call details
   ↓
8. Call information returned to framework
   ↓
9. PhoneApp receives intent
   ↓
10. InCallUI displays incoming call screen
    ↓
11. User answers/rejects call
```

### 3. Data Connection Establishment

```
1. ConnectivityService requests data connection
   ↓
2. DataConnection state machine activated
   ↓
3. DcTracker.setupData()
   ↓
4. RILJ.setupDataCall()
   ↓
5. IRadio.setupDataCall() - Radio HAL
   ↓
6. Vendor RIL sends QMI_WDS_START_NETWORK_INTERFACE_REQ
   ↓
7. Modem establishes PDN connection
   ↓
8. QMI_WDS_START_NETWORK_INTERFACE_RESP received
   ↓
9. Data call parameters returned (IP, DNS, MTU)
   ↓
10. Network interface configured (rmnet_data0)
    ↓
11. Routing tables updated
    ↓
12. ConnectivityService notified
    ↓
13. Apps can use data connection
```

---

## Data Flow Architecture

### Data Path Components

```
┌──────────────────────────────────────────┐
│  Applications                            │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  TCP/IP Stack (Linux Kernel)             │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  Network Interface (rmnet_data0-7)       │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  RmNet Driver (MAP Protocol)             │
│  - Multiplexing                          │
│  - QoS handling                          │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  IPA (Integrated Packet Accelerator)     │
│  - Hardware acceleration                 │
│  - Packet filtering                      │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  Modem Processor                         │
│  - Protocol stack (LTE/5G)               │
│  - Radio transmission                    │
└──────────────────────────────────────────┘
```

### QMI Services Used

1. **QMI_DMS (Device Management)**
   - Device info
   - Operating mode
   - Power management

2. **QMI_NAS (Network Access)**
   - Network registration
   - Signal strength
   - Cell selection

3. **QMI_WDS (Wireless Data)**
   - Data connection setup
   - Packet statistics
   - QoS management

4. **QMI_VOICE**
   - Call control
   - DTMF
   - Supplementary services

5. **QMI_UIM (User Identity Module)**
   - SIM card access
   - Authentication
   - File operations

6. **QMI_IMS (IP Multimedia Subsystem)**
   - VoLTE calls
   - Video calls
   - RCS messaging

---

## Technical Deep Dive: Key Files

### AOSP Source Locations

```
frameworks/opt/telephony/
├── src/java/com/android/internal/telephony/
│   ├── Phone.java
│   ├── GsmCdmaPhone.java
│   ├── RIL.java
│   ├── PhoneFactory.java
│   └── dataconnection/DcTracker.java

hardware/interfaces/radio/
├── 1.6/
│   ├── IRadio.hal
│   ├── IRadioResponse.hal
│   └── IRadioIndication.hal

hardware/ril/
├── libril/
│   ├── ril.cpp
│   ├── ril_service.cpp
│   └── ril_commands.h
└── rild/
    └── rild.c
```

### Qualcomm Vendor Files (Typical Locations)

```
vendor/qcom/proprietary/
├── qcril-hal/
│   ├── qcril_qmi/
│   │   ├── qcril_qmi_client.c
│   │   ├── qcril_qmi_voice.c
│   │   ├── qcril_qmi_nas.c
│   │   └── qcril_qmi_ims.c
│   └── modules/
│       ├── voice/
│       ├── data/
│       └── nas/

vendor/qcom/proprietary/qmi/
├── services/
│   ├── voice_service_v02.c
│   ├── network_access_service_v01.c
│   └── wireless_data_service_v01.c
```

---

## Debugging & Development

### Key Tools

1. **logcat**
   ```bash
   adb logcat -b radio
   adb logcat -s RILJ:V
   ```

2. **QXDM (Qualcomm eXtensible Diagnostic Monitor)**
   - Modem log analysis
   - Protocol stack debugging
   - RF measurements

3. **AT Commands**
   ```bash
   adb shell
   echo -e "AT+CGMI\r" > /dev/smd0
   ```

4. **getprop/setprop**
   ```bash
   adb shell getprop | grep radio
   adb shell setprop persist.radio.apm_sim_not_pwdn 1
   ```

5. **dumpsys**
   ```bash
   adb shell dumpsys telephony.registry
   adb shell dumpsys phone
   ```

### Important Log Tags

- **RILJ**: RIL Java layer
- **RIL**: Native RIL daemon
- **QCRIL**: Qualcomm RIL
- **QMI**: QMI framework
- **Telephony**: Framework telephony
- **IMS**: IMS stack
- **DataConnection**: Data call management

---

## Security Considerations

### Trust Boundaries

```
┌────────────────────────────────┐
│  Untrusted Apps                │  ← SELinux: untrusted_app
├────────────────────────────────┤
│  System Apps                   │  ← SELinux: system_app
├────────────────────────────────┤
│  Framework (system_server)     │  ← SELinux: system_server
├────────────────────────────────┤
│  Radio HAL (hwservicemanager)  │  ← SELinux: hal_radio
├────────────────────────────────┤
│  Vendor RIL (rild)             │  ← SELinux: rild
├────────────────────────────────┤
│  Kernel Drivers                │  ← Kernel space
└────────────────────────────────┘
```

### Permission Model

- **READ_PHONE_STATE**: Basic phone info
- **CALL_PHONE**: Initiate calls
- **MODIFY_PHONE_STATE**: System-level access (signature)
- **READ_PRIVILEGED_PHONE_STATE**: Privileged access (signature)

---

## Performance Optimization

### Fast Dormancy
- Reduces power consumption
- Quick transition between active/idle
- Qualcomm-specific implementation

### Carrier Aggregation
- Combines multiple LTE carriers
- Modem handles aggregation
- AOSP receives aggregated data

### IPA (Integrated Packet Accelerator)
- Hardware offload engine
- Reduces CPU usage
- Direct modem-to-apps data path

---

## Summary

### Key Takeaways

1. **Layered Architecture**
   - Clear separation of concerns
   - Framework → HAL → Vendor → Modem

2. **QMI is Critical**
   - Message-based protocol
   - Standardized service interfaces
   - Transport agnostic

3. **HAL Provides Abstraction**
   - HIDL/AIDL interfaces
   - Version management
   - Process isolation

4. **Vendor Implementation**
   - Qualcomm provides vendor RIL
   - Closed-source modem firmware
   - QMI client implementation

5. **Multiple Subsystems**
   - Modem runs independent OS
   - IPC for communication
   - Hardware acceleration where possible

---

## References

- AOSP Source: https://android.googlesource.com/
- Radio HAL: hardware/interfaces/radio/
- Telephony Framework: frameworks/opt/telephony/
- Qualcomm IPC Router: drivers/soc/qcom/
- Android Compatibility Definition Document (CDD)
- Radio HAL Documentation: source.android.com

---

## Questions & Discussion

Topics for deeper exploration:
- 5G NR architecture differences
- VoLTE/IMS implementation details
- eSIM and DSDS (Dual SIM) architecture
- Modem firmware updates and security
- Carrier customization mechanisms
