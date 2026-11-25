# Lab 5: From Initial Access to RRC Connection Setup (Student Guide)
 
## 1. Lab Goal
 
In Lab 1, you observed UE registration with the 5GC and the establishment of data service. This lab moves the observation point earlier: before registering with the Core network, how does a UE find a gNB, obtain cell system information, perform Random Access, and finally establish an RRC connection?
 
The main procedure in this lab is:
 
```text
Cell search / initial synchronization
→ MIB / SIB1
→ Msg1 PRACH
→ Msg2 RAR
→ Msg3 RRCSetupRequest
→ Msg4 RRCSetup
→ RRCSetupComplete
→ RRC_CONNECTED
```
 
This lab ends at `RRC_CONNECTED`. NAS Registration, Authentication, and PDU Session establishment belong to the Core network scope covered in Lab 1.
 
## 2. Evidence Used in This Lab
 
You will receive three files produced by the same successful OAI run:
 
| File | Purpose |
|---|---|
| `Lab5-ue.log` | Observe UE cell search, synchronization, PRACH, and RRC state. |
| `Lab5-gnb.log` | Cross-check gNB-side behavior when needed. |
| `Lab5.pcap` | Use Wireshark to observe MIB, SIB1, RAR, and RRC messages. |
 
You are not required to start OAI for this lab. Your main task is to reconstruct and explain the access procedure using the supplied log and PCAP files. Section 7 provides optional steps for reproducing the files yourself.
 
## 3. Learning Objectives
 
After completing this lab, you should be able to:
 
1. Explain the UE procedure from Cell Search to `RRC_CONNECTED` in the correct order.
2. Distinguish the roles of broadcast system information, Random Access, and RRC Connection Setup.
3. Obtain complementary evidence from a UE log and a Wireshark MAC-NR trace.
4. Explain the meaning of `RRC_CONNECTED` and how it differs from 5GC Registration.
## 4. Open the Supplied Files
 
Perform all analysis from the `openairinterface5g` repository root. Put the three supplied files in `lab5-artifacts/`, or in another location of your choice. If you use another location, adjust the paths in the commands accordingly.
 
```text
lab5-artifacts/Lab5-ue.log
lab5-artifacts/Lab5-gnb.log
lab5-artifacts/Lab5.pcap
```
 
Open `lab5-artifacts/Lab5.pcap` in Wireshark.
 
If Wireshark shows only UDP packets or the `nr-rrc` filter does not work, complete the following setup once:
 
1. Open **Analyze → Enabled Protocols...**, search for, and enable `MAC-NR-FRAMED`.
2. Open **Preferences → Protocols → MAC-NR** and enable NR RRC decoding for BCCH/CCCH data.
3. Open **Preferences → Protocols → RLC-NR** and enable RRC decoding for CCCH data and AM reassembly.
4. Close and reopen the PCAP.
> **Setup used:** The three files were placed in `~/openairinterface5g-v2.4.0/lab5-artifacts/`, and all `grep` commands were run from the repository root. In Wireshark, `MAC-NR-FRAMED` was enabled and RRC decoding was turned on for MAC-NR and RLC-NR. After reopening the file, all 5135 packets decode as MAC-NR / RLC-NR / PDCP-NR / NR RRC (see the screenshots below).
 
## 5. Submission and Grading
 
Add a brief explanation below every screenshot. You do not need to draw boxes or arrows on screenshots.
 
### Task 1: From Cell Search to Msg2 RAR — 35 points
 
Complete the following four steps in order. For each step, provide the requested values or screenshots and briefly explain what the evidence shows and why it matters in the access procedure.
 
#### Step 1: Find Cell Search and Initial Synchronization — 8 points
 
Run the following command on `Lab5-ue.log`:
 
```bash
grep -nE 'Starting cell search|Initial sync successful' lab5-artifacts/Lab5-ue.log
```
 
Record whether UE synchronization succeeded and the PCI value. Explain: after completing initial synchronization, what is the UE able to do?
 
![task1-step1](https://hackmd.io/_uploads/rkcoxpmy7e.png)

 
```text
Starting cell search      : line 56  (center freq 3619200000 Hz, 106 PRBs, 1 GSCN scanned)
Initial sync successful   : line 63
Synchronization succeeded : Yes
PCI                       : 0
```
 
**Explanation:** Line 56 shows the UE starting to scan for an SSB at 3619.2 MHz. Line 63 confirms that it found one and synchronized to the cell, whose Physical Cell ID is **PCI = 0**. In between, the log shows that the PBCH was decoded successfully (line 58) and that a carrier frequency offset of only 2 Hz was measured and corrected (line 62).
 
**What the UE can do after initial synchronization:** The UE is now aligned to the cell in **time** (it knows the frame, slot and symbol boundaries) and in **frequency**. It also knows the **PCI**, which it needs to descramble the downlink channels. This lets it decode the PBCH to read the **MIB**, and then use the MIB to find and decode **SIB1**. It can also measure the cell's signal strength (RSRP) for cell selection. It **cannot transmit yet**: it doesn't know the RACH configuration, and it has no uplink timing alignment.
 
#### Step 2: Find MIB and SIB1 — 8 points
 
In Wireshark, enter:
 
```text
nr-rrc
```
 
Find one `MIB` packet and one `SIB1` packet. Record the frame number of each and provide screenshots.
 
MIB and SIB1 are repeatedly broadcast system-information messages. The UE must obtain the common configuration before it can continue the access procedure. Briefly explain the purpose of MIB and SIB1 for UE access.
 
```text
MIB  frame number = 2
SIB1 frame number = 1
```
 
**MIB (frame 2)**
 
![task1-step2-mib](https://hackmd.io/_uploads/B1e6epQyml.png)

 
**Explanation:** Frame 2 is a `MAC-NR BCCH PDU` on the **BCH transport channel**, which is carried on the PBCH. It contains the MIB. Its key fields are:
- `systemFrameNumber`: the SFN bits the UE needs to know the frame timing.
- `subCarrierSpacingCommon: scs30or120`: the subcarrier spacing used for SIB1 and the initial access messages (30 kHz here).
- `pdcch-ConfigSIB1` (`controlResourceSetZero: 11`, `searchSpaceZero: 0`): tells the UE **where to look on the PDCCH for SIB1**.
- `cellBarred: notBarred`: the UE is allowed to camp on this cell.
**SIB1 (frame 1): cell access information**
 
![task1-step2-sib-a](https://hackmd.io/_uploads/rk6ReTQ17l.png)

 
**Explanation:** Frame 1 is a `MAC-NR BCCH PDU` on the **DL-SCH**, addressed with **SI-RNTI**. It carries SIB1. `cellAccessRelatedInfo` contains:
- the **PLMN ID, MCC 208 / MNC 95**,
- the **trackingAreaCode** 0x00a000 (40960),
- the **cellIdentity** 12345678.
The UE uses these to check whether this cell belongs to a network it is allowed to use.
 
**SIB1 (frame 1): Random Access configuration**
 
![task1-step2-sib-b](https://hackmd.io/_uploads/SJelbT7JQl.png)

 
**Explanation:** In the same SIB1, `servingCellConfigCommon → uplinkConfigCommon → initialUplinkBWP → rach-ConfigCommon` holds the common Random Access settings:
- `prach-ConfigurationIndex: 98`: when PRACH occasions occur.
- `preambleReceivedTargetPower: -104 dBm`, `powerRampingStep: dB2` and `preambleTransMax: n10`: preamble power and retries.
- `ra-ResponseWindow: sl20`: how long to wait for Msg2.
- `ra-ContentionResolutionTimer: sf64`: how long to wait for Msg4.
- `prach-RootSequenceIndex` and `msg1-SubcarrierSpacing: kHz30`: which preamble sequences to use.
**Purpose of MIB and SIB1 for UE access:**
- **MIB** is the minimum information broadcast on the PBCH. It gives the UE the frame timing and subcarrier spacing, says whether the cell is barred, and tells it where to find SIB1.
- **SIB1** says whether the UE may access this cell (PLMN, TAC, cell ID). It also gives the common uplink and RACH configuration (PRACH occasions, preamble power, RAR window, contention resolution timer) and the TDD pattern.
Without SIB1, the UE would not know when, where or how to send Msg1, so it could not start Random Access. Both messages repeat throughout the capture (frames 1/2, 3/4, 5/6, …). SIB1 appears before MIB here only because both were logged in the same slot (SFN 68, slot 0) by the tracer. Logically, the UE must decode the MIB first.
 
#### Step 3: Find Msg1 PRACH — 8 points
 
Run the following command on the UE log:
 
```bash
grep -nE 'PRACH' lab5-artifacts/Lab5-ue.log
```
 
Record the Msg1 log line number. From the line that shows the actual PRACH transmission, find:
 
```text
frame.slot = 553.19
preambleIndex = 16
```
 
**Msg1 log line number = 81**
 
![task1-step3-a](https://hackmd.io/_uploads/HyYbbTmJml.png)

 
**Explanation:** The grep returns three lines:
- **Line 79** is the MAC selecting a PRACH occasion (slot 19, start symbol 4).
- **Line 80** is the PRACH scheduler choosing the occasion in frame 553, slot 19.
- **Line 81** is the PHY actually transmitting the preamble: `PRACH [UE 0] in frame.slot 553.19 … preambleIndex = 16`. This is the Msg1 line.
Next, return to Wireshark. Clear the display filter, press `Cmd/Ctrl + F`, choose **Packet details** as the search scope, and search for:
 
```text
RACH Preamble chosen
```
 
After locating the PRACH event recorded by the gNB tracer, record:
 
```text
RAPID = 16   (frame 489)
```
 
This packet is displayed as `MAC-LTE RACH Preamble chosen`; it is the OAI tracer MAC-event format.
 
![task1-step3-b](https://hackmd.io/_uploads/H1nfZTQJ7g.png)

 
**Explanation:** Frame 489 is the gNB tracer's PRACH event, `RACH Preamble chosen for UE 0 (RAPID=16, attempt=0)`, with `[RAPID: 16]` and `RACH Attempt Number: 0`. It is shown in the MAC-LTE event format, so its `[System Frame Number: 554] [Subframe: 3]` context is the tracer's own timestamp, not the NR frame.slot. The gNB log gives the NR timing: `553.19 Initiating RA procedure with preamble 16` (gnb.log line 186), which matches the UE log.
 
Submit one UE-log screenshot and one Wireshark screenshot. Briefly answer:
 
1. Which Random Access message is this PRACH transmission?
   > **Answer:** It is **Msg1**, the Random Access preamble, sent by the UE on the PRACH. It is the first message of the 4-step contention-based Random Access (CBRA) procedure (`Initialization of 4-Step CBRA procedure`, UE log line 78).
2. At which `frame.slot` and with which `preambleIndex` does the UE log show transmission?
   > **Answer:** At **frame.slot 553.19** (frame 553, slot 19), with **preambleIndex = 16** (UE log line 81).
3. Does the Wireshark `RAPID` match the UE-log `preambleIndex`? What does this show?
   > **Answer:** **Yes. RAPID = 16 = preambleIndex.** The gNB detected exactly the preamble that the UE transmitted, so Msg1 was received successfully. The gNB can now answer that specific preamble in the RAR. Two independent sources, the UE log and the gNB-side trace, confirm the same event.
#### Step 4: Find Msg2 RAR in Wireshark — 11 points
 
Return to Wireshark and clear the display filter. Press `Cmd/Ctrl + F`, choose **Packet details** as the search scope, and search for `RAR` or `Random Access Response`. After locating the packet, record its frame number and find the following values in `MAC-NR RAR`:
 
```text
Frame number = 490
RAPID = 16
Temporary C-RNTI = 0xf21e (61982)
```
 
![task1-step4](https://hackmd.io/_uploads/H1HVbp7yQg.png)

 
**Explanation:** Frame 490 is the `MAC-NR RAR`, the Random Access Response (**Msg2**). It is sent on the downlink at SFN 554, slot 10 and addressed with **RA-RNTI 0x010f (271)**. The RA-RNTI is derived from the PRACH occasion the UE used, so only UEs that transmitted in that occasion decode this RAR. It contains:
- a **Backoff Indicator subheader** (BI = 0, 5 ms), which only matters if access has to be retried;
- a **RAPID subheader** with **RAPID = 16**. It matches the UE's preambleIndex, so this RAR answers *our* UE's preamble. The UE log confirms it: `Found RAR with the intended RAPID 16` at [554.10], line 85.
That subheader gives the UE three things:
- **Timing Advance = 0**, which aligns the UE's uplink timing. It is 0 because the RF simulator has no propagation delay.
- an **uplink Grant** (`0x02e6016`: Msg3 PUSCH frequency allocation 742, time allocation 0, MCS 1, TPC 0 dB), the resources to send Msg3.
- **Temporary C-RNTI = 0xf21e (61982)**, the temporary identity the UE uses from now on.
The gNB log shows the same values: `RA-RNTI 010f TC-RNTI f21e … RAPID 16` (line 187), `Msg3 scheduled at 554.19` (line 189), and `Generating RA-Msg2 DCI … preamble_index(RAPID) 16` (line 190).
 
### Task 2: Complete Random Access and RRC Connection Setup — 40 points
 
Complete the following steps in order. For each message, provide a screenshot and record the requested frame number, fields, and brief explanation.
 
#### Step 1: Find RRCSetupRequest — 10 points
 
In Wireshark, use:
 
```text
nr-rrc.rrcSetupRequest_element
```
 
Expand `UL-CCCH-Message → rrcSetupRequest` and find:
 
```text
Frame number = 491
Context RNTI = 0xf21e (61982)
```
 
![task2-step1](https://hackmd.io/_uploads/H1uHW6Xymg.png)

 
**Explanation:** Frame 491 is a `MAC-NR UL-SCH` PDU sent at SFN 554, slot 19. That matches the UE log line 86 (`[554.19] RA-Msg3 transmitted`) and the Msg3 slot the gNB scheduled. Its MAC subheader carries **LCID 52 (CCCH, 48 bits)**, so the RRC message is sent on **SRB0 / UL-CCCH** without any security. The `rrcSetupRequest` contains:
- `ue-Identity: randomValue` (39 bits), a random identity the UE chose because it has no 5G-S-TMSI yet;
- `establishmentCause: mo-Signalling`, meaning a mobile-originated signalling connection for registration.
Briefly answer:
 
1. Does the `Context RNTI` match the `Temporary C-RNTI` in the Task 1 / Step 4 RAR? What does this show?
   > **Answer:** **Yes.** The Context RNTI is **0xf21e (61982)**, the same as the Temporary C-RNTI given in the RAR (frame 490). The UE sent Msg3 on the uplink grant from the RAR, using the TC-RNTI that the RAR assigned. This links Msg3 to the same Random Access attempt as Msg1 (preamble 16) and Msg2.
2. Can you claim that the RRC connection has been established after seeing only `RRCSetupRequest`? Why or why not?
   > **Answer:** **No.** `RRCSetupRequest` is only the UE's *request*. At this point:
   > - **contention is not yet resolved**: another UE could have chosen the same preamble 16 in the same occasion, received the same RAR and sent its own Msg3 on the same grant;
   > - the gNB has not yet accepted the request or sent any configuration;
   > - no SRB1 exists, and the UE is still in **RRC_IDLE**.
   >
   > The connection is established only after the UE receives **RRCSetup (Msg4)** with a matching contention resolution identity. The UE then enters RRC_CONNECTED and confirms with RRCSetupComplete.
#### Step 2: Find RRCSetup — 10 points
 
Clear the previous filter and enter:
 
```text
nr-rrc.rrcSetup_element
```
 
Expand `DL-CCCH-Message → rrcSetup` and find:
 
```text
Frame number = 492
Context RNTI = 0xf21e (61982)
rrc-TransactionIdentifier = 1
```
 
Check whether `Contention Resolution` also appears in the packet.
 
> **Contention Resolution present: Yes.** The MAC PDU is `MAC-NR DL-SCH (Contention Resolution) (LCID:0 139 bytes)`, with `LCID: UE Contention Resolution Identity (62)` and `UE Contention Resolution Identity: 1c8e65ea60e6`.
 
![task2-step2](https://hackmd.io/_uploads/Bk-wWp71me.png)

 
**Explanation:** Frame 492 is sent on the downlink at SFN 555, slot 10 to RNTI **0xf21e**. This matches the UE log line 87 `[555.10] 4-Step RA procedure succeeded`. The single MAC PDU contains two subPDUs:
1. **UE Contention Resolution Identity MAC CE** (LCID 62), value `1c8e65ea60e6`. This is the first 48 bits of the UE's Msg3 CCCH SDU, the RRCSetupRequest, echoed back by the gNB.
2. **CCCH SDU** (LCID 0, 139 bytes), carrying `DL-CCCH-Message → rrcSetup` with **rrc-TransactionIdentifier 1**. Its `criticalExtensions` contain `radioBearerConfig`, which sets up **SRB1**, and `masterCellGroup`, which holds the RLC, MAC and physical-layer configuration.
Below the screenshot, briefly answer:
 
1. Which message is this in four-step Random Access?
   > **Answer:** **Msg4**, the contention resolution message of the 4-step Random Access procedure. Here it also carries the RRCSetup.
2. What does it mean that `Contention Resolution` and `RRCSetup` appear in the same downlink message?
   > **Answer:** One downlink transmission completes two procedures at once:
   > - **MAC layer: Random Access is finished.** The UE compares the Contention Resolution Identity (`1c8e65ea60e6`) with the Msg3 it sent. They match, so the UE knows it **won contention**, and its Temporary C-RNTI 0xf21e becomes its **C-RNTI**. A UE that lost contention would see a different identity and discard the message, including the RRCSetup.
   > - **RRC layer: the connection is set up.** The gNB accepts the RRCSetupRequest and immediately sends the configuration (SRB1, cell group config) the UE needs for the RRC connection.
   >
   > Because the RRCSetup is tied to the contention resolution identity, only the UE that sent that exact Msg3 applies the RRC configuration. The UE log shows this order: RA succeeded (line 87), then `Received NR_RRCSetup` (line 88).
#### Step 3: Find RRCSetupComplete — 12 points
 
Clear the previous filter and enter:
 
```text
nr-rrc.rrcSetupComplete_element
```
 
Select the packet and first verify that `MAC-NR UL-SCH` appears at the top. Expand:
 
```text
RLC-NR → PDCP-NR → NR Radio Resource Control (RRC) protocol
  → UL-DCCH-Message → message: c1
```
 
Expand `message: c1` further and confirm that it contains `rrcSetupComplete`. Record:
 
```text
Frame number = 498
Context RNTI = 0xf21e (61982)
rrc-TransactionIdentifier = 1
```
 
![task2-step3](https://hackmd.io/_uploads/SJGO-TQkQl.png)

 
**Explanation:** Frame 498 has `MAC-NR UL-SCH` at the top (uplink, SFN 557, slot 17, RNTI 0xf21e, **RNTI Type: C-RNTI**). Unlike Msg3, it is not sent on CCCH. It goes through the full signalling stack:
- MAC subheader **LCID 1**,
- `RLC-NR … [AM] SRB:1`: acknowledged mode on SRB1,
- `PDCP-NR (SN=0)`: SRB-1, signalling plane,
- `UL-DCCH-Message → c1: rrcSetupComplete`.
This proves that the UE is using the **SRB1** configured by the RRCSetup. The message also carries `selectedPLMN-Identity: 1` and a `dedicatedNAS-Message` containing the NAS **Registration request**, which is the hand-over point to the 5GC procedure covered in Lab 1.
 
Below the screenshot, briefly answer:
 
1. Does the `Context RNTI` still match the `Temporary C-RNTI` in the RAR?
   > **Answer:** **Yes, it is still 0xf21e (61982).** Wireshark now labels it `RNTI Type: C-RNTI`: after contention resolution succeeded, the Temporary C-RNTI from the RAR became the UE's permanent C-RNTI. The same identity runs through Msg2 (frame 490), Msg3 (frame 491), Msg4 (frame 492) and RRCSetupComplete (frame 498), so all of these messages belong to the same UE and the same access attempt.
2. Does the `rrc-TransactionIdentifier` match the `RRCSetup` in Step 2? What is the purpose of this comparison?
   > **Answer:** **Yes, both are 1.** The RRC transaction identifier pairs a response with the request that triggered it. A matching value shows that this RRCSetupComplete answers *that specific* RRCSetup (frame 492). The RRC setup transaction is therefore closed correctly, and the gNB can be sure the UE applied that configuration. The gNB log confirms it: `Received RRCSetupComplete (RRC_CONNECTED reached)` at line 205.
#### Step 4: Confirm the Final Random Access and RRC Connection Result — 8 points
 
Run the following command on the UE log:
 
```bash
grep -nE '4-Step RA procedure succeeded|Received NR_RRCSetup|RRCSetupComplete|State = NR_RRC_CONNECTED' \
  lab5-artifacts/Lab5-ue.log
```
 
From the output, record:
 
```text
4-Step RA procedure succeeded: line 87
Received NR_RRCSetup: line 88
State = NR_RRC_CONNECTED: line 90
Generating RRCSetupComplete: line 93
```
 
![task2-step4](https://hackmd.io/_uploads/BkHK-TmJQl.png)

 
**Explanation:** The four lines show, in order, the end of Random Access and the start of the RRC connection on the UE side, all with RNTI 0xf21e:
- line 87: `[555.10] 4-Step RA procedure succeeded. CBRA: Contention Resolution is successful.`
- line 88: `Logical Channel DL-CCCH (SRB0), Received NR_RRCSetup`
- line 90: `State = NR_RRC_CONNECTED`
- line 93: `Logical Channel UL-DCCH (SRB1), Generating RRCSetupComplete`
The timing in the UE log matches the Wireshark trace exactly:
 
| Event | UE log | Wireshark |
|---|---|---|
| Msg1 PRACH | 553.19 (line 81) | frame 489, RAPID 16 |
| Msg2 RAR | 554.10 (line 85) | frame 490, SFN 554 slot 10 |
| Msg3 RRCSetupRequest | 554.19 (line 86) | frame 491, SFN 554 slot 19 |
| Msg4 RRCSetup + Contention Resolution | 555.10 (line 87–88) | frame 492, SFN 555 slot 10 |
| RRCSetupComplete | line 93 | frame 498, SFN 557 slot 17 |
 
Submit a screenshot of this log output and briefly answer:
 
1. In log order, what are the three events from successful Random Access to generating `RRCSetupComplete`?
   > **Answer:**
   > 1. **Random Access succeeds** (line 87): contention is resolved by Msg4.
   > 2. **RRCSetup is received** on DL-CCCH / SRB0 (line 88), and SRB1 is added (line 89).
   > 3. **The UE enters RRC_CONNECTED** (line 90).
   >
   > After these three events, the UE **generates RRCSetupComplete** on UL-DCCH / SRB1 (line 93), with the NAS Registration Request inside (line 92). Note that the UE enters RRC_CONNECTED when it applies the RRCSetup, *before* it sends RRCSetupComplete. The gNB considers the UE connected only when the Complete arrives (gNB log line 205).
2. What do `4-Step RA procedure succeeded` and `State = NR_RRC_CONNECTED` each prove?
   > **Answer:**
   > - **`4-Step RA procedure succeeded`** proves the **MAC layer** procedure is complete. The gNB detected the preamble, the UE received the RAR with timing advance and a grant, its Msg3 was received, and it **won contention**. The UE now has uplink synchronization and its own **C-RNTI (0xf21e)**. It does *not* yet prove an RRC connection exists.
   > - **`State = NR_RRC_CONNECTED`** proves the **RRC layer** connection is established. The UE has an RRC context with the gNB and a dedicated signalling bearer (**SRB1**) for exchanging RRC and NAS messages.
   >
   > Neither line means the UE is registered with the 5GC. RRC_CONNECTED is a connection between the UE and the **gNB** only. 5GC Registration (NAS) starts afterwards, with the Registration Request in RRCSetupComplete, followed by Authentication (UE log line 95), Security Mode and Registration Accept. That is Lab 1's scope.
### Task 3: Draw an Initial Access MSC Diagram — 25 points
 
Using the log and Wireshark evidence found in Tasks 1 and 2, draw your own Message Sequence Chart (MSC). It must include two vertical lifelines, `UE` and `gNB`, and the following six messages: MIB/SIB1, Msg1, Msg2, Msg3, Msg4, and `RRCSetupComplete`.
 
Determine and draw the correct order and direction of the messages yourself. Add a one-sentence purpose next to each arrow.
 
![Untitled (1)](https://hackmd.io/_uploads/HJvyfT7y7g.png)

 
**Reading the diagram:**
- **Lifelines:** `UE` on the left, `gNB` on the right; time runs downward.
- **UE states:** the UE starts in **RRC_IDLE** and enters **RRC_CONNECTED** after receiving and applying Msg4 (RRCSetup), before it sends RRCSetupComplete.
- **Direction:** the downlink arrows (gNB → UE) are MIB/SIB1 (dashed, because it is a broadcast to all UEs), Msg2 and Msg4. The uplink arrows (UE → gNB) are Msg1, Msg3 and RRCSetupComplete.
- **Order:** MIB/SIB1 → Msg1 → Msg2 → Msg3 → Msg4 → RRCSetupComplete, matching the UE log line order (lines 63 → 71 → 81 → 85 → 86 → 87/88 → 90 → 93) and the Wireshark frame order (2/1 → 489 → 490 → 491 → 492 → 498).

| Item | Points |
|---|---:|
| UE and gNB lifelines, with the two UE states identified | 5 |
| Complete message order: MIB/SIB1, Msg1, Msg2, Msg3, Msg4, RRCSetupComplete | 12 |
| Correct direction for every message arrow | 5 |
| Reasonable brief purpose for every major stage | 3 |
| **Task 3 subtotal** | **25** |
 
| Item | Points |
|---|---:|
| Task 1: Initial Access evidence timeline | 35 |
| Task 2: Key-message analysis | 40 |
| Task 3: Initial Access MSC diagram | 25 |
| **Total** | **100** |
 
## 7. Optional: Reproduce the Log and PCAP Files
 
This section is not required for the assignment. If you want to generate the same kind of evidence using your own OAI run, follow these steps in an OAI/5GC environment that was already set up in an earlier class.
 
The commands below assume that you are currently in the `openairinterface5g` repository root. Use three terminals and first create a directory for the artifacts:
 
```bash
mkdir -p lab5-artifacts
```
 
### Terminal A: Start the gNB and Wait for the Tracer
 
```bash
cd cmake_targets/ran_build/build
 
sudo -E env RFSIMULATOR=server ./nr-softmodem \
  -O ../../../ci-scripts/conf_files/gnb.sa.band78.106prb.rfsim.conf \
  --rfsim --T_stdout 2 \
  2>&1 | tee ../../../lab5-artifacts/Lab5-gnb.log
```
 
### Terminal B: Record the gNB MAC/RRC Trace
 
```bash
cd cmake_targets/ran_build/build
 
./common/utils/T/tracer/record \
  -d ../../../common/utils/T/T_messages.txt \
  -o ../../../lab5-artifacts/Lab5-gnb-wireshark.raw \
  -on WIRESHARK
```
 
The gNB in Terminal A continues after the tracer connects.
 
### Terminal C: Start the UE and Save the UE Log
 
```bash
cd cmake_targets/ran_build/build
 
sudo -E RFSIMULATOR=127.0.0.1 ./nr-uesoftmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
  --rfsim -r 106 --numerology 1 --band 78 -C 3619200000 \
  2>&1 | tee ../../../lab5-artifacts/Lab5-ue.log
```
 
After `State = NR_RRC_CONNECTED` appears, wait a few seconds, then stop the terminals in this order: recorder in Terminal B, UE in Terminal C, and gNB in Terminal A.
 
### Convert the Trace to a Wireshark PCAP
 
```bash
cd cmake_targets/ran_build/build
 
./common/utils/T/tracer/macpdu2wireshark \
  -d ../../../common/utils/T/T_messages.txt \
  -i ../../../lab5-artifacts/Lab5-gnb-wireshark.raw \
  -to-file ../../../lab5-artifacts/Lab5.pcap
```
 
You should obtain:
 
```text
lab5-artifacts/Lab5-ue.log
lab5-artifacts/Lab5-gnb.log
lab5-artifacts/Lab5.pcap
```
 
> **Note:** This optional section was not performed. All analysis in this report uses the supplied `Lab5-ue.log`, `Lab5-gnb.log` and `Lab5.pcap` from the same successful OAI run.
 
