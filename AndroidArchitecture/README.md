# Android AOSP & Qualcomm Modem Architecture - Technical Presentation

This directory contains comprehensive technical documentation about Android AOSP architecture and its interaction with Qualcomm modem subsystems.

## Contents

### 1. Presentation Document
**File:** `AOSP_Qualcomm_Modem_Architecture.md`

A complete technical presentation covering:
- AOSP architecture overview (layer by layer)
- Qualcomm modem subsystem components
- Radio Interface Layer (RIL) architecture
- Hardware Abstraction Layer (HAL)
- QMI (Qualcomm MSM Interface) protocol
- Call flow examples (voice, data)
- Data path architecture
- Security considerations
- Debugging tools and techniques

### 2. Draw.io Diagrams

#### Diagram 1: Overall AOSP Architecture
**File:** `1_AOSP_Overall_Architecture.drawio`

Visual representation of:
- Complete Android stack from applications to hardware
- Application layer (Phone, Messaging, Settings apps)
- Framework layer (TelephonyManager, ActivityManager, etc.)
- System services and native libraries
- HAL layer with all major HAL modules
- Linux kernel components
- Hardware layer including Qualcomm SoC and modem

#### Diagram 2: AOSP-Qualcomm Modem Interaction
**File:** `2_AOSP_Qualcomm_Modem_Interaction.drawio`

Detailed interaction diagram showing:
- Complete communication path from Android framework to modem
- Left side: Android stack (Framework → RILJ → HAL → Vendor RIL → QMI)
- Right side: Modem subsystem (QMI services → Protocol stack → DSP → RF)
- IPC mechanisms (IPC Router, SMD, RmNet drivers)
- QMI service interfaces
- Modem processor architecture
- Bi-directional communication flows

#### Diagram 3: RIL Architecture & Call Flows
**File:** `3_RIL_Architecture_CallFlows.drawio`

Three comprehensive call flow diagrams:

**A. Outgoing Voice Call Flow (Solicited Request)**
- Step-by-step flow from user dialing to modem processing
- Response callback chain
- QMI message structure for DIAL request

**B. Incoming Call Flow (Unsolicited Indication)**
- Network-initiated call setup
- Unsolicited indication propagation
- Query mechanism for call details
- UI notification flow

**C. Data Connection Setup Flow**
- Complete PDN connection establishment
- RmNet driver and IPA (Packet Accelerator) integration
- Network interface configuration
- Data path from apps to network

## How to Use

### Viewing the Presentation
1. Open `AOSP_Qualcomm_Modem_Architecture.md` in any Markdown viewer or editor
2. For best experience, use a Markdown preview tool that supports GitHub-flavored markdown
3. The document contains code blocks, diagrams (ASCII art), and detailed explanations

### Opening Draw.io Diagrams

**Option 1: Online (Recommended)**
1. Go to https://app.diagrams.net (formerly draw.io)
2. Click "Open Existing Diagram"
3. Select any of the `.drawio` files from this directory
4. The diagram will open in the web-based editor

**Option 2: Desktop Application**
1. Download draw.io desktop app from https://github.com/jgraph/drawio-desktop/releases
2. Install and open the application
3. Open any `.drawio` file from this directory

**Option 3: VS Code Extension**
1. Install the "Draw.io Integration" extension in VS Code
2. Open any `.drawio` file directly in VS Code
3. Edit and view diagrams within the editor

### Diagram Features
- **Zoom:** Use mouse wheel or zoom controls
- **Pan:** Click and drag the canvas
- **Export:** File → Export as (PNG, JPEG, PDF, SVG)
- **Edit:** Modify any component, add notes, change colors
- **Print:** File → Print for presentation handouts

## Key Concepts Covered

### AOSP Architecture
- Layered architecture design
- Separation of concerns
- Binder IPC mechanism
- Project Treble and vendor interface

### Qualcomm Modem Integration
- Modem subsystem (MSS) architecture
- QMI protocol and services
- IPC Router vs SMD communication
- AMSS (Advanced Mobile Subscriber Software)

### RIL (Radio Interface Layer)
- RILJ (Java layer) implementation
- Radio HAL (HIDL/AIDL interfaces)
- Vendor RIL (Qualcomm QCRIL)
- Solicited vs Unsolicited responses

### Data Flow
- PDN (Packet Data Network) setup
- RmNet multiplexing driver
- IPA (Integrated Packet Accelerator)
- End-to-end data path

### QMI Services
- QMI_VOICE: Call control
- QMI_NAS: Network access
- QMI_WDS: Wireless data
- QMI_UIM: SIM operations
- QMI_IMS: VoLTE/IMS

## Technical Depth

This presentation is designed for:
- Android platform engineers
- Modem integration engineers
- Telephony framework developers
- System architects
- Technical leads and senior engineers

**Prerequisites:**
- Understanding of Android architecture
- Familiarity with Linux kernel concepts
- Knowledge of cellular network basics (LTE/5G)
- Experience with C++ and Java

## Source Code References

Key AOSP source locations mentioned:
```
frameworks/opt/telephony/          # Telephony framework
hardware/interfaces/radio/         # Radio HAL definitions
hardware/ril/                      # Native RIL daemon
```

Qualcomm vendor files (typical locations):
```
vendor/qcom/proprietary/qcril-hal/ # Qualcomm RIL HAL
vendor/qcom/proprietary/qmi/       # QMI services
```

## Debugging Commands

Quick reference for debugging:
```bash
# Radio logs
adb logcat -b radio

# RILJ debug
adb logcat -s RILJ:V

# Telephony info
adb shell dumpsys telephony.registry
adb shell dumpsys phone

# Radio properties
adb shell getprop | grep radio
```

## Additional Resources

- AOSP Source: https://android.googlesource.com/
- Radio HAL Documentation: https://source.android.com/devices/tech/connect
- Android CDD: https://source.android.com/compatibility/cdd
- Qualcomm Developer Network: https://developer.qualcomm.com/

## Presentation Tips

For presenting this material:

1. **Start with Overview**: Use Diagram 1 to show overall architecture
2. **Deep Dive**: Use Diagram 2 to explain modem interaction
3. **Concrete Examples**: Use Diagram 3 to walk through actual call flows
4. **Reference Document**: Use the MD file for detailed explanations

**Suggested Flow:**
- 5 min: AOSP overview (Diagram 1)
- 10 min: Modem architecture and QMI (Diagram 2)
- 15 min: Call flow examples (Diagram 3)
- 10 min: Q&A and deep dives

## License

This documentation is created for educational and technical reference purposes.

## Version

Created: January 2026
Format: Markdown + Draw.io XML
Compatibility: AOSP 11+ (Android R and above)

---

For questions or updates, please refer to the official AOSP documentation and Qualcomm developer resources.
