# SUPL Presentation Materials

This directory contains comprehensive presentation materials for **SUPL (Secure User Plane Location)** based on the OMA SUPL 2.0 specification and Broadcom white paper.

## Files Created

### 1. PowerPoint Presentation Content
**File:** `SUPL_Presentation_Outline.md`

A complete presentation outline with 20 slides covering:
- Introduction to SUPL and location services
- Architecture and components (SET, SLP, SLC, SPC)
- Operating modes (Proxy and Non-Proxy)
- Positioning protocols (RRLP, RRC, TIA-801, LPP)
- SUPL 2.0 enhancements
- Benefits and use cases
- Security features
- Industry adoption

**How to use:** Convert this markdown content into your PowerPoint presentation. Each `## Slide X` section represents one slide with title and content.

---

### 2. Draw.io Diagrams

#### A. Architecture Diagram
**File:** `SUPL_Architecture.drawio`

**Shows:**
- Complete SUPL architecture
- SET (SUPL Enabled Terminal) components
- SLP (SUPL Location Platform) with SLC and SPC
- Communication protocols (ULP, ILP, MLP)
- LCS Client interface
- Color-coded components for easy understanding

**Key elements:**
- Blue: SET components
- Orange: SLC components
- Green: SPC components
- Pink: LCS Client
- Protocol connections with labels

---

#### B. SUPL INIT Process (Network-Initiated)
**File:** `SUPL_INIT_Process.drawio`

**Shows:**
- Complete network-initiated positioning flow (Proxy Mode)
- Message sequence from LCS Client to SET
- All SUPL messages: INIT, POS INIT, POS, END
- TLS connection setup
- Positioning session details
- Position calculation and delivery

**10 Steps:**
1. MLP SLIR (Location Request)
2. SUPL INIT
3. TCP/TLS Connection Setup
4. SUPL POS INIT
5. ILP to SPC
6. SUPL POS (Positioning Session)
7. ILP Response
8. SUPL END
9. Connection Teardown
10. MLP SLIA (Location Response)

---

#### C. SUPL START Process (SET-Initiated)
**File:** `SUPL_START_Process.drawio`

**Shows:**
- Complete SET-initiated positioning flow (Non-Proxy Mode)
- Application requesting location on device
- SUPL START and RESPONSE messages
- Assistance data delivery
- Position calculation (SET-Based or SET-Assisted)
- Location delivery to application

**9 Steps:**
1. Application requests location
2. TCP/TLS Connection Setup
3. SUPL START
4. SUPL RESPONSE
5. SUPL POS INIT
6. ILP Assistance Data Request
7. SUPL POS (Positioning)
8. SUPL END
9. Location delivered to app

**Use Cases:** Navigation apps, LBS applications, user-initiated positioning

---

#### D. SUPL 2.0 Triggered Positioning
**File:** `SUPL_Triggered_Positioning.drawio`

**Shows:**
- SUPL 2.0 triggered positioning feature
- Session setup with trigger parameters
- Multiple location reports based on triggers
- Periodic reporting flow
- Session termination

**Trigger Types:**
1. **Periodic:** Reports at fixed intervals
2. **Area Event:** Entry/exit from geographic area
3. **Start Time:** Begin reporting at specific time
4. **Stop Time:** Maximum session duration
5. **Historic Reporting:** Buffered locations when offline

**Use Cases:**
- Fleet management (track vehicles every 5 minutes)
- Geo-fencing (alerts on area entry/exit)
- Asset tracking (periodic shipment updates)
- Child safety (continuous location monitoring)
- Emergency services (ongoing location during emergency)

---

## Source Documents

The materials are based on:
1. **OMA-AD-SUPL-V2_0-20120417-A.pdf** - Official OMA SUPL 2.0 Architecture Document (April 2012)
2. **SUPL-WP100-R.pdf** - Broadcom SUPL White Paper (October 2007)

---

## How to Use These Materials

### For PowerPoint Presentation:
1. Open `SUPL_Presentation_Outline.md`
2. Copy content into PowerPoint slides
3. Import the draw.io diagrams as images
4. Customize styling and branding

### For Draw.io Diagrams:
1. Open each `.drawio` file in draw.io (https://app.diagrams.net/)
2. Edit as needed
3. Export as PNG, SVG, or PDF for presentation
4. Insert into PowerPoint slides

**Recommended diagram placement in presentation:**
- Slide 5-6: `SUPL_Architecture.drawio`
- Slide 9: `SUPL_INIT_Process.drawio`
- Slide 10: `SUPL_START_Process.drawio`
- Slide 11: `SUPL_Triggered_Positioning.drawio`

---

## Quick Reference

### SUPL Components
- **SET**: SUPL Enabled Terminal (mobile device)
- **SLP**: SUPL Location Platform (network server)
  - **SLC**: SUPL Location Center (session management, security)
  - **SPC**: SUPL Positioning Center (position calculation)
- **LCS Client**: Location services application

### Protocols
- **ULP**: UserPlane Location Protocol (SET ↔ SLP)
- **ILP**: Internal Location Protocol (SLC ↔ SPC)
- **MLP**: Mobile Location Protocol (LCS Client ↔ SLC)

### Positioning Protocols
- **RRLP**: Radio Resource LCS Protocol (GSM)
- **RRC**: Radio Resource Control (UMTS)
- **TIA-801**: IS-801 positioning (CDMA)
- **LPP**: LTE Positioning Protocol (LTE)

### Operating Modes
- **Proxy Mode**: Network-initiated (LCS Client → SLP → SET)
- **Non-Proxy Mode**: SET-initiated (SET → SLP)

---

## Tips for Presentation

1. **Start with the problem**: Explain Control Plane limitations before introducing SUPL
2. **Use the architecture diagram early**: Help audience visualize components
3. **Walk through message flows**: Use sequence diagrams to explain processes step-by-step
4. **Highlight SUPL 2.0 benefits**: Emphasize triggered positioning as a key enhancement
5. **Use real-world examples**: Fleet management, emergency services, navigation
6. **Address security**: Show TLS encryption and authentication mechanisms
7. **Compare with alternatives**: SUPL vs Control Plane advantages

---

## Presentation Time Estimates

- **15 minutes**: Slides 1-10 (Introduction + Architecture + Basic Flow)
- **25 minutes**: Slides 1-15 (Add SUPL 2.0 + Benefits)
- **35 minutes**: Full presentation with Q&A

---

## Additional Resources

- OMA SUPL Specifications: www.openmobilealliance.org
- 3GPP Location Services: www.3gpp.org
- Draw.io: https://app.diagrams.net/

---

**Created:** December 28, 2025
**Based on:** OMA SUPL 2.0 Specification & Broadcom White Paper
**Format:** Markdown + Draw.io XML