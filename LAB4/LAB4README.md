# Lab 4 — 5G CU/DU Split Deployment and F1-C/F1-U Analysis

**Student handout**  
**Environment:** Ubuntu 24.04 LTS · OpenAirInterface (OAI) v2.4.0 · RFsimulator · no SDR required  
**Prerequisite:** Lab 3 completed successfully (OAI CN5G, gNB, nrUE, UE tunnel and Ping)

> **Lab objective:** Replace the monolithic gNB from Lab 3 with a split gNB-CU and gNB-DU. Capture and analyze both **F1-C** (F1AP over SCTP) and **F1-U** (GTP-U over UDP) in a single PCAP file. Verify that the UE can still establish a PDU session and reach the external data network.
>
> **Instructor validation:** This lab's CU/DU split, F1 Setup, UE RRC signaling, UE Context procedures, GTP-U T-PDUs, and external-DN Ping were observed on the classroom OAI v2.4.0 environment. Exact IP addresses, frequencies and core network configuration must be checked on each student image.

---

## 1. Learning objectives

After completing this lab, you should be able to:

1. Explain the functional division between gNB-CU and gNB-DU.
2. Deploy a split 5G SA RAN with OAI and RFsimulator.
3. Identify **F1-C** control-plane messages (F1 Setup, RRC transfer, UE Context Setup).
4. Identify **F1-U** user-plane GTP-U packets, including their direction and TEID.
5. Verify UE IP connectivity through the split architecture.
6. Distinguish F1-U from the separate **N3** interface, even though both can use GTP-U.

## 2. Network architecture and experimental parameters

```text
                         OAI 5G Core
                     AMF / SMF / UPF
                           |
                   N2 (NGAP) / N3 (GTP-U)
                           |
                      +----------+
                      |  gNB-CU  |  F1-C IP: 127.0.0.3
                      | RRC/PDCP |  (example classroom setup)
                      +----------+
                         |     |
        F1-C: F1AP/SCTP  |     | F1-U: GTP-U/UDP
                         |     |
                      +----------+
                      |  gNB-DU  |  F1 IP: 127.0.0.4
                      |RLC/MAC/PHY|
                      +----------+
                           |
                    RFsimulator (IQ/TCP)
                           |
                         nrUE
                           |
                      oaitun_ue1
                           |
                       ICMP Ping
```

| Parameter | Classroom configuration |
|---|---|
| OAI version | `v2.4.0` |
| RAN | One gNB-CU + one gNB-DU + one nrUE |
| Radio | RFsimulator, no physical SDR |
| NR band | n78 |
| Bandwidth | 106 PRBs |
| Numerology | 1 (30 kHz SCS) |
| CU F1 address (observed) | `127.0.0.3` |
| DU F1 address (observed) | `127.0.0.4` |
| DU SSB NR-ARFCN (observed) | `630048` (3,450.72 MHz) |
| UE frequency argument used in successful split test | `-C 3450720000` |
| UE IMSI | `001010000000001` |
| External DN | `192.168.70.135` |

**Important:** The Lab 3 monolithic gNB config used `absoluteFrequencySSB = 641280` (3,619.20 MHz), whereas the split DU config used `630048` (3,450.72 MHz). The frequency parameters **must match the working DU/UE pair**. Do not reuse Lab 3's `-C 3619200000` without checking the DU configuration. The `-C` argument and effective radio carrier placement should be verified against OAI startup logs; the above value is the one used in the successful classroom test, not a general formula for every DU file.

---

## 3. Task 1 — Inspect the split configurations

### Step 1. Confirm the configuration files exist

```bash
cd ~/openairinterface5g-v2.4.0

ls -l \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf
```

### Step 2. Check frequency and F1 network settings

```bash
grep -nE \
'absoluteFrequencySSB|dl_absoluteFrequencyPointA|dl_carrierBandwidth|physCellId|min_rxtxtime|local_s_address|remote_s_address|local_n_address|remote_n_address|local_s_portd|remote_s_portd|local_n_portd|remote_n_portd|amf_ip_address' \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf \
  targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf
```

**Record:** The CU and DU F1 addresses, the DU SSB frequency, and PRB count. If the local settings differ from the table above, adapt the following commands and capture filters accordingly.

![task1](https://hackmd.io/_uploads/rkE0qq5AGx.png)


**Record:**

| Item | CU (`gnb-cu.sa.f1.conf`) | DU (`gnb-du.sa.band78.106prb.rfsim.pci0.conf`) |
|---|---|---|
| F1 local address | `local_s_address = 127.0.0.3` | `local_n_address = 127.0.0.4` |
| F1 remote address | `remote_s_address = 0.0.0.0` (accepts any DU) | `remote_n_address = 127.0.0.3` (points to CU) |
| F1-U ports | `local_s_portd = 2152`, `remote_s_portd = 2152` | `local_n_portd = 2152`, `remote_n_portd = 2152` |
| AMF IP | `192.168.70.132` | — |
| Physical cell ID | — | `physCellId = 0` |
| SSB frequency | — | `absoluteFrequencySSB = 630048` |
| PointA | — | `dl_absoluteFrequencyPointA = 628776` |
| PRB count | — | `dl_carrierBandwidth = 106` |
| Timing | — | `min_rxtxtime = 4` |

**Frequency check.** For NR-ARFCN between 600000 and 2016666, F = 3000 MHz + 15 kHz × (N − 600000):

| Quantity | Calculation | Result |
|---|---|---|
| SSB | 3000 + 0.015 × 30048 | **3450.72 MHz** |
| PointA | 3000 + 0.015 × 28776 | 3431.64 MHz |
| Carrier bandwidth | 106 PRB × 12 × 30 kHz | 38.16 MHz |
| Carrier centre | 3431.64 + 38.16 / 2 | **3450.72 MHz** |

The DU startup log confirms the calculation and even prints the matching UE command line:

```text
[RRC]    Read in ServingCellConfigCommon (PhysCellId 0, ABSFREQSSB 630048, DLBand 78, ABSFREQPOINTA 628776, DLBW 106, ...
[RRC]    absoluteFrequencySSB 630048 corresponds to 3450720000 Hz
[NR_MAC] Command line parameters for OAI UE: -C 3450720000 -r 106 --numerology 1 --band 78 --ssb 516
```


**Questions**

1. **Which protocol layers reside in the CU, and which in the DU?** The gNB-CU hosts the upper layers: RRC (control plane), PDCP and SDAP. The gNB-DU hosts the lower layers: RLC, MAC and PHY. This is the 3GPP "Option 2" (higher-layer) split.
2. **What is the purpose of the F1 interface?** F1 is the standardized interface between gNB-CU and gNB-DU. F1-C (F1AP over SCTP) sets up and manages the CU–DU link and per-UE contexts and carries RRC messages inside RRC containers. F1-U (GTP-U over UDP) carries user-plane PDCP PDUs for each data radio bearer.
3. **Why must the UE frequency match the DU rather than Lab 3?** The DU owns the PHY, so it decides where the SSB is transmitted (630048 → 3450.72 MHz). The UE must search at that frequency to detect PSS/SSS, decode the PBCH/MIB and then SIB1. With the Lab 3 value it would scan 3619.2 MHz and never find the cell.

---

## 4. Task 2 — Deploy the split 5G SA network

Before starting, stop the monolithic Lab 3 gNB and any old nrUE processes with `Ctrl+C`. Avoid running two gNB/DU RFsimulator servers on the same listening port.

### Terminal 1 — Start CN5G

```bash
cd ~/oai-cn5g
docker compose up -d
docker compose ps
```
![task2-core](https://hackmd.io/_uploads/SJpbiccCGl.png)


All CN5G containers (mysql, ims, nrf, udr, udm, ausf, amf, smf, upf, ext-dn) are up. The screenshot was taken about 10 s after `docker compose up -d`, while health checks were still starting. The successful NG Setup and UE registration below confirm that the core became fully operational.
### Terminal 2 — Start gNB-CU

```bash
cd ~/openairinterface5g-v2.4.0/cmake_targets/ran_build/build

sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-cu.sa.f1.conf
```

![task2-cu](https://hackmd.io/_uploads/rkt4s55Azx.png)


The CU completes NG Setup with the AMF, opens its F1 SCTP socket on 127.0.0.3, and accepts the DU (`Received F1 Setup Request from gNB_DU 3584 (du-rfsim)` → `sending F1 Setup Response`). Relevant CU log lines for the two GTP-U endpoints:

```text
[GTPU]   Configuring GTPu address : 192.168.70.129, port : 2152     <- N3 side (towards UPF)
[F1AP]   F1AP_CU_SCTP_REQ(create socket) for 127.0.0.3 len 10       <- F1-C
[GTPU]   Initializing UDP for local address 127.0.0.3 with port 2152 <- F1-U side (towards DU)
[NR_RRC] Bearer Context Setup: PDU Session ID=10, incoming TEID=0x00000001, Addr=192.168.70.134
```

### Terminal 3 — Start gNB-DU

**Start the packet capture in Task 3 *before* starting the DU if you want to record the initial F1 Setup handshake.**

```bash
cd ~/openairinterface5g-v2.4.0/cmake_targets/ran_build/build

sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-du.sa.band78.106prb.rfsim.pci0.conf \
  --rfsim
```
![task2-du](https://hackmd.io/_uploads/HyEwo95Cfl.png)


The DU binds F1-C and GTP-U to 127.0.0.4, connects to the CU at 127.0.0.3, receives the F1 Setup Response and the gNB-DU Configuration Update Acknowledge, and then starts the RFsimulator server.
### Terminal 4 — Start nrUE

```bash
cd ~/openairinterface5g-v2.4.0/cmake_targets/ran_build/build

sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3450720000 \
  --uicc0.imsi 001010000000001 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1
```

> **Instructor note:** This UE command reflects the working classroom split configuration. In particular, the original Lab 3 `-C 3619200000` setting was not compatible with this DU configuration. If a provided classroom image has different NR frequencies or UICC credentials, use the instructor-validated values.

![task2-ue](https://hackmd.io/_uploads/H1Jsscc0fl.png)


The UE connects to the RFsimulator server (127.0.0.1:4043), finds the cell at 3450.72 MHz and synchronizes to PCI 0. The rest of the UE log shows registration and PDU session establishment:

```text
[PHY]    Initial sync successful, PCI: 0
[NR_RRC] SIB1 decoded
[MAC]    [UE 0][RAPROC] 4-Step RA procedure succeeded. CBRA: Contention Resolution is successful.
[NR_RRC] State = NR_RRC_CONNECTED
[NAS]    Generate Initial NAS Message: Registration Request
[NAS]    Received Registration Accept with result 3GPP
[NAS]    Send NAS_UPLINK_DATA_REQ message(PduSessionEstablishRequest)
[NAS]    Received PDU Session Establishment Accept, UE IPv4: 10.0.0.2
[OIP]    Interface oaitun_ue1 successfully configured, IPv4 10.0.0.2, IPv6 (null)
```

### Check the UE connection

```bash
ip -4 addr show oaitun_ue1
ping -I oaitun_ue1 -c 10 192.168.70.135
```

![task2-ping](https://hackmd.io/_uploads/HJk6s9qCMg.png)


**Record:**

| Item | Observed value |
|---|---|
| UE IP (`oaitun_ue1`) | **10.0.0.2/24** |
| Packets transmitted / received | **10 / 10** |
| Packet loss | **0 %** |
| RTT min / avg / max / mdev | 14.228 / **22.144** / 27.228 / 3.736 ms |

The replies arrive with TTL 63 instead of 64, which shows that one routing hop (the UPF) sits between the UE tunnel and the external DN.

**Evidence:** Screenshots of CU/DU startup, UE connection, and successful Ping. Record the observed UE IP, transmitted/received packets, packet loss, and average RTT. Do **not** invent values; they vary by machine.

---

## 5. Task 3 — Capture F1-C and F1-U in a single PCAP

The classroom capture showed CU `127.0.0.3` and DU `127.0.0.4` on the Linux loopback interface (`lo`).

### Step 1. Start packet capture

Start CN5G and CU, but **before starting the DU** open another terminal:

```bash
sudo tcpdump -i lo -s 0 -nn \
  -w ~/lab4-f1-complete.pcap \
  'host 127.0.0.3 and host 127.0.0.4 and (sctp or udp port 2152)'
```

This captures:

- **F1-C:** F1AP carried over SCTP.
- **F1-U:** GTP-U carried over UDP port 2152, **provided the configured F1-U endpoints and port match these values**.

### Step 2. Trigger control-plane and user-plane traffic

1. Start the DU to trigger **SCTP association** and **F1 Setup**.
2. Start nrUE to trigger **RRC** and **UE Context** procedures.
3. After the UE tunnel appears, generate traffic:

```bash
ping -I oaitun_ue1 -c 30 -i 0.2 192.168.70.135
```

4. Press `Ctrl+C` in the **tcpdump terminal** to finish and save the PCAP.
5. Verify the file:

```bash
ls -lh ~/lab4-f1-complete.pcap
```

### Alternative: capture all traffic between CU and DU

If you are unsure of the F1-U UDP port, use this **instead of** the filtered capture above:

```bash
sudo tcpdump -i lo -s 0 -nn \
  -w ~/lab4-f1-all.pcap \
  'host 127.0.0.3 and host 127.0.0.4'
```

If the CU and DU are not communicating over `lo`, select the correct interface (`ip -br addr` or `tcpdump -D`) and update the IP filter.

![task3-terminal](https://hackmd.io/_uploads/SygEn9qAzg.png)


**Record:**

| Item | Value |
|---|---|
| Interface | `lo` (CU and DU both use loopback addresses) |
| CU / DU addresses in filter | 127.0.0.3 / 127.0.0.4 |
| Protocol filter | `sctp or udp port 2152` (F1-U port taken from the configs) |
| Packets captured | 96 (0 dropped by kernel) |
| File | `/home/ubuntu/lab4-f1-complete.pcap`, 16 K |

![task3](https://hackmd.io/_uploads/BkgS29qCMe.png)


| Content of the PCAP | Display filter | Packets |
|---|---|---:|
| F1-C (SCTP, including 24 F1AP messages) | `sctp` | 33 |
| F1-U (GTP-U T-PDUs) | `gtp` | 63 |
| **Total** | | **96** |


---

## 6. Task 4 — Analyze F1-C (F1AP / SCTP)

Open the capture:

```bash
wireshark ~/lab4-f1-complete.pcap
```

Useful **Wireshark display filters**:

```text
sctp
f1ap
```

Find and record these messages:

| Message | Typical direction | Meaning |
|---|---|---|
| SCTP INIT / INIT_ACK / COOKIE_ECHO / COOKIE_ACK | DU ↔ CU | Establish SCTP association |
| F1 Setup Request | DU → CU | DU initiates F1 connection and provides served-cell information |
| F1 Setup Response | CU → DU | CU accepts F1 Setup |
| GNBDUConfigurationUpdate | DU → CU | DU informs CU of configuration changes |
| Initial UL RRC Message Transfer (RRC Setup Request) | DU → CU | UE initial RRC request forwarded to CU |
| DL RRC Message Transfer (RRC Setup) | CU → DU | CU delivers RRC Setup toward UE |
| UL RRC Message Transfer (RRC Setup Complete) | DU → CU | UE completes RRC procedure; NAS Registration Request can be carried inside |
| UE Context Setup Request / Response | CU ↔ DU | Establish UE-specific resources/context |
| DL RRC Message Transfer (RRC Reconfiguration) | CU → DU | CU sends further RRC configuration |

**Observed classroom sequence:** The successful capture contained SCTP association, F1 Setup Request/Response, DU Configuration Update/Acknowledgement, RRC Setup messages, Authentication signaling carried through RRC, UE Context Setup/Modification, and RRC Reconfiguration. The exact packet numbers and timing are **not fixed**.
###  Recorded F1-C messages

| No. | Time (s) | Source → Destination | Message |
|---:|---:|---|---|
| 1 | 0.000000 | 127.0.0.4 → 127.0.0.3 | SCTP INIT |
| 2 | 0.000168 | 127.0.0.3 → 127.0.0.4 | SCTP INIT_ACK |
| 3 | 0.000205 | 127.0.0.4 → 127.0.0.3 | SCTP COOKIE_ECHO |
| 4 | 0.000266 | 127.0.0.3 → 127.0.0.4 | SCTP COOKIE_ACK |
| 5 | 0.015572 | 127.0.0.4 → 127.0.0.3 | **F1SetupRequest** (MIB, SIB1) |
| 7 | 0.017205 | 127.0.0.3 → 127.0.0.4 | **F1SetupResponse** |
| 9 | 0.017660 | 127.0.0.4 → 127.0.0.3 | GNBDUConfigurationUpdate |
| 10 | 0.018118 | 127.0.0.3 → 127.0.0.4 | GNBDUConfigurationUpdateAcknowledge |
| 12 | 21.450922 | 127.0.0.4 → 127.0.0.3 | **InitialULRRCMessageTransfer** — RRC Setup Request |
| 13 | 21.452847 | 127.0.0.3 → 127.0.0.4 | **DLRRCMessageTransfer** — RRC Setup |
| 14 | 21.473633 | 127.0.0.4 → 127.0.0.3 | **ULRRCMessageTransfer** — RRC Setup Complete + NAS Registration request |
| 27 | 21.997712 | 127.0.0.3 → 127.0.0.4 | **UEContextSetupRequest** |
| 28 | 22.000007 | 127.0.0.4 → 127.0.0.3 | **UEContextSetupResponse** |
| 29 | 22.013870 | 127.0.0.3 → 127.0.0.4 | **DLRRCMessageTransfer** — RRC Reconfiguration |
| 30 | 22.032552 | 127.0.0.4 → 127.0.0.3 | ULRRCMessageTransfer — RRC Reconfiguration Complete |
| 31 / 32 | 22.033 / 22.034 | CU → DU / DU → CU | UEContextModificationRequest / Response |

Frames 15–25 are further UE-associated RRC transfers carrying the NAS and AS procedures between RRC Setup and UE Context Setup:

| No. | Direction | Content |
|---:|---|---|
| 15 / 16 | CU → DU / DU → CU | NAS Authentication request / response |
| 17 / 18 | CU → DU / DU → CU | NAS Security mode command / complete |
| 19 / 20 | CU → DU / DU → CU | RRC (AS) Security Mode Command / Complete |
| 21 / 22 | CU → DU / DU → CU | UE Capability Enquiry / Information |
| 23–25 | CU → DU, DU → CU | DL/UL Information Transfer (integrity-protected NAS: Registration Accept, Registration Complete, PDU Session Establishment Request) |

Frames 17 and 19 are both "security mode commands" but at different layers. Frame 17 is NAS security between UE and AMF; frame 19 is RRC/AS security between UE and CU. Both are carried transparently over F1-C.


### SCTP association

![task4-SCTP association](https://hackmd.io/_uploads/SJZR29cCzg.png)


The DU (127.0.0.4, port 37617) opens the association to the CU's F1-C port **38472** with the four-way handshake INIT → INIT_ACK → COOKIE_ECHO → COOKIE_ACK (2 outbound / 2 inbound streams).

### All F1AP messages

![task4-F1AP messages](https://hackmd.io/_uploads/HJv7T59AGe.png)


### F1 Setup Request and Response

![task4-F1 Setup Request](https://hackmd.io/_uploads/BkKNpq5CGl.png)


![task4-F1 Setup Response](https://hackmd.io/_uploads/BkmBTqqAzg.png)


| Frame | Direction | Key IEs |
|---|---|---|
| 5 | DU 127.0.0.4 → CU 127.0.0.3 | `procedureCode: id-F1Setup (1)`, TransactionID 1, **gNB-DU-ID 3584**, **gNB-DU-Name du-rfsim**, gNB-DU-Served-Cells-List (with MIB/SIB1), gNB-DU-RRC-Version |
| 7 | CU 127.0.0.3 → DU 127.0.0.4 | `successfulOutcome`, TransactionID 1, **gNB-CU-Name cu-rfsim**, **Cells-to-be-Activated-List** (NR-CGI of the DU's cell) |

### UE-associated F1AP messages

![task4-UE-associated messages (2)](https://hackmd.io/_uploads/HymL6ccRze.png)


Frame 12 (InitialULRRCMessageTransfer) carries **gNB-DU-UE-F1AP-ID 44351**, the **NR-CGI** (MCC 001, MNC 01, cell identity 12345678) and **C-RNTI 44351** (`0xad3f`). From frame 14 onwards the messages also carry the CU's **gNB-CU-UE-F1AP-ID 1**. This pair of IDs is what makes them UE-associated signaling.

### NAS Registration Request inside F1AP (frame 14)

![task4-NAS inside F1AP-a](https://hackmd.io/_uploads/ByPwa95CMe.png)


![task4-NAS inside F1AP-b](https://hackmd.io/_uploads/SyM_accCGe.png)


```text
F1AP ULRRCMessageTransfer (procedureCode 13)
 ├─ gNB-CU-UE-F1AP-ID 1, gNB-DU-UE-F1AP-ID 44351, SRBID 1
 └─ RRCContainer
     └─ PDCP-NR (SRB-1, uplink, SN 0)
         └─ RRC UL-DCCH: rrcSetupComplete
             └─ dedicatedNAS-Message
                 └─ 5GS NAS: Registration request (0x41), plain NAS
```

### UE Context Setup (frames 27 and 28)

![task4-UE Context Setup Request-a](https://hackmd.io/_uploads/S17969cCze.png)


![task4-UE Context Setup Request-b](https://hackmd.io/_uploads/Hk356c50Me.png)


![task4-UE Context Setup Response](https://hackmd.io/_uploads/HkM3a59AMl.png)


| Frame | Direction | Content |
|---|---|---|
| 27 | CU → DU | gNB-CU-UE-F1AP-ID 1, gNB-DU-UE-F1AP-ID 44351, SpCell-ID, CUtoDURRCInformation, SRBs-ToBeSetup, **DRBs-ToBeSetup** (DRB QoS 5QI 9, S-NSSAI SST 1, QoS flow QFI 1), **UL UP TNL: 127.0.0.3, gTP-TEID `7dc29161`**, gNB-DU-UE-AMBR-UL |
| 28 | DU → CU | DUtoCURRCInformation (CellGroupConfig), **DRBs-Setup-List: DRB ID 1, LCID 4, DL UP TNL: 127.0.0.4, gTP-TEID `815b5aed`** |

These two TEIDs are exactly the ones observed on the F1-U packets in Task 5.

**Questions**

1. **Which node initiates F1 Setup, and which acknowledges it?** The **gNB-DU** initiates it: F1SetupRequest in frame 5 from 127.0.0.4 to 127.0.0.3. The **gNB-CU** acknowledges it with F1SetupResponse in frame 7. This matches the configuration: the DU has `remote_n_address = 127.0.0.3`, while the CU only listens (`remote_s_address = 0.0.0.0`).
2. **What is the difference between SCTP association establishment and F1 Setup?** SCTP association (frames 1–4) is a transport-layer handshake. It builds a reliable, multi-stream connection between two IP/port endpoints, and its cookie mechanism protects against SYN-flood-style attacks. It knows nothing about 5G. F1 Setup (frames 5 and 7) is an application-layer F1AP procedure that runs over that association. The DU identifies itself (ID 3584, name du-rfsim), reports its served cell with MIB/SIB1 and RRC version, and the CU replies with its name and the list of cells to activate. SCTP must succeed first, and F1 Setup follows about 15 ms later.
3. **How can a NAS Registration Request travel over F1-C?** By nesting (frame 14). The UE puts the NAS Registration Request into the `dedicatedNAS-Message` of RRC Setup Complete. That RRC message, protected by PDCP on SRB1, is carried as an opaque **RRC-Container** inside an F1AP ULRRCMessageTransfer. The DU does not decode RRC or NAS; it only relays the bytes. The CU's RRC extracts the NAS message and forwards it to the AMF over NGAP (N2).
4. **What is the role of UE Context Setup?** It creates the UE's lower-layer context in the DU for the new bearers (SRB2 and DRB1). In frame 27 the CU sends the DRB list with QoS and its uplink F1-U endpoint (127.0.0.3, TEID `7dc29161`). In frame 28 the DU confirms the DRB (LCID 4), returns its CellGroupConfig (RLC/MAC/PHY configuration) and its downlink F1-U endpoint (127.0.0.4, TEID `815b5aed`). The CU then builds the RRC Reconfiguration (frame 29) from that information. UE Context Setup therefore also establishes the F1-U tunnel.
5. **Does F1 Setup alone prove successful UE registration?** **No.** F1 Setup completed at 0.017 s, while the first UE message (frame 12) only arrived at 21.45 s, before any UE existed. F1 Setup only proves that the CU–DU link is up and the cell is activated. Registration is proven separately: by the NAS exchange in frames 14–25, the `Registration Accept` and `PDU Session Establishment Accept, UE IPv4: 10.0.0.2` lines in the UE log, and the successful ping.
---

## 7. Task 5 — Analyze F1-U (GTP-U / UDP)

In the **same PCAP**, apply:

```text
gtp
```

If necessary, first check:

```text
udp.port == 2152
```

Directional display filters (based on the observed classroom CU/DU IP pair):

```text
gtp && ip.src == 127.0.0.4 && ip.dst == 127.0.0.3
gtp && ip.src == 127.0.0.3 && ip.dst == 127.0.0.4
```

In the captured GTP-U packet details, inspect:

```text
Internet Protocol (outer CU/DU addresses)
  User Datagram Protocol
    GPRS Tunneling Protocol (GTP-U)
      Message Type: G-PDU / T-PDU
      Tunnel Endpoint Identifier (TEID)
      Encapsulated user payload (if decodable)
```

**Important:** The classroom trace showed repeated `GTP T-PDU` packets from `127.0.0.4` (DU) to `127.0.0.3` (CU), confirming traffic on the CU–DU user-plane path. The actual TEID was **not recorded** in the provided screenshot: read the TEID from your own packet details instead of assuming a value.

### Distinguish F1-U from N3

Both **F1-U** and **N3** can use GTP-U. Do **not** classify traffic as F1-U solely because Wireshark says `GTP` or the port is `2152`.

- **F1-U:** GTP-U between **gNB-DU and gNB-CU**.
- **N3:** GTP-U between **gNB-CU and 5GC UPF**.

Identify the interface using the actual source/destination endpoints and configuration.



**Evidence:** Screenshot of at least one GTP-U T-PDU, with outer IP addresses, UDP ports, message type, and TEID visible. Record the observed UL and DL packet counts (if both directions are present). Do not assume equal counts.

### All F1-U traffic

![task5-All F1-U traffic](https://hackmd.io/_uploads/rkjGCc9RMg.png)


All 63 F1-U packets are GTP-U **T-PDUs** between 127.0.0.4 and 127.0.0.3 on UDP 2152. The ping traffic (frames 37–96) alternates between uplink (137 bytes) and downlink (149 bytes) every 0.2 s.

### Uplink T-PDU (frame 37)

![task5-One uplink T-PDU](https://hackmd.io/_uploads/SkejmAc5AMx.png)


### Downlink T-PDU (frame 38)

![task5-One downlink T-PDU](https://hackmd.io/_uploads/HJO4A59RGl.png)


**Record:**

| Item | Uplink (frame 37) | Downlink (frame 38) |
|---|---|---|
| Outer IP | 127.0.0.4 → 127.0.0.3 | 127.0.0.3 → 127.0.0.4 |
| UDP ports | 2152 → 2152 | 2152 → 2152 |
| GTP flags | `0x30` (no extension header) | `0x34` (extension header present) |
| Message type | **T-PDU (0xff)** | **T-PDU (0xff)** |
| **TEID** | **`0x7dc29161`** (2109903201) | **`0x815b5aed`** (2170247917) |
| Extension header | — | NR RAN container (`0x84`) |
| Frame length | 137 bytes | 149 bytes |
| **Packet count** | **33** | **30** |

**TEID cross-check with F1-C:**

| Direction | Announced over F1-C | Used on F1-U |
|---|---|---|
| UL (DU → CU) | Frame 27 UL UP TNL: 127.0.0.3, `7dc29161` | Frame 37 TEID `0x7dc29161` |
| DL (CU → DU) | Frame 28 DL UP TNL: 127.0.0.4, `815b5aed` | Frame 38 TEID `0x815b5aed` |

The 12 extra bytes on downlink packets are the GTP-U NR RAN container extension header (NR-U user-plane information from CU to DU). Uplink packets have none.

### Encapsulated payload

![task5-The payload-frame33-ul](https://hackmd.io/_uploads/SymLCq9RGe.png)


![task5-The payload-frame38-dl](https://hackmd.io/_uploads/By9IAc9Cfg.png)


Wireshark shows the payload only as raw `T-PDU Data` because F1-U carries **PDCP PDUs**, not plain IP packets. Ciphering is `nea0` (null), so the contents can be read from the hex:

| Frame | PDCP header | Inner packet |
|---:|---|---|
| 33 (UL) | `80 00 00` (data PDU, SN 0) | IPv6 `60…`, next header `3a` (ICMPv6), `fe80::a407:aae1:30d8:6567` → `ff02::2`, type `85` = **Router Solicitation** |
| 37 (UL) | `80 00 03` (SN 3) | IPv4 `45…`, TTL 64, ICMP, 10.0.0.2 → 192.168.70.135, type `08` = **echo request** |
| 38 (DL) | `80 00 00` (SN 0) | IPv4 `45…`, TTL 63, ICMP, 192.168.70.135 → 10.0.0.2, type `00` = **echo reply** |

### UL and DL packet counts

![task5-UL and DL counts-a](https://hackmd.io/_uploads/r1QOCq5CMl.png)


![task5-UL and DL counts-b](https://hackmd.io/_uploads/B1K_0qqCMl.png)


Statistics → Conversations → UDP (not limited to a display filter): **A (127.0.0.4) → B (127.0.0.3) = 33 packets (UL)**, **B → A = 30 packets (DL)**, total 63, over 23.5 s.

The counts are **not equal**. The 30-packet ping produced 30 echo requests and 30 echo replies. The 3 extra uplink packets (frames 33, 35 and 36, at 22.1 s, 26.4 s and 35.1 s) are IPv6 Router Solicitations sent by the UE's Linux kernel when `oaitun_ue1` came up. Nothing answered them. The PDCP sequence numbers confirm this: the Router Solicitations used UL SNs 0–2, so the first echo request (frame 37) has SN 3, while the first downlink packet (frame 38) has SN 0.

### F1-C vs F1-U vs N3

| | F1-C | F1-U | N3 |
|---|---|---|---|
| Endpoints | gNB-DU ↔ gNB-CU | gNB-DU ↔ gNB-CU | gNB-CU ↔ UPF |
| Addresses in this lab | 127.0.0.4 ↔ 127.0.0.3 | 127.0.0.4 ↔ 127.0.0.3 | 192.168.70.129 ↔ 192.168.70.134 |
| Protocol stack | F1AP / SCTP (38472) | GTP-U / UDP (2152) | GTP-U / UDP (2152) |
| Carries | RAN control signaling + RRC containers | PDCP PDUs of a DRB | Plain user IP packets (PDU Session Container with QFI) |
| Tunnels/IDs set up by | — | F1AP UE Context Setup (frames 27/28) | NGAP PDU Session Resource Setup |
| Present in this PCAP? | Yes | Yes | No (runs on the Docker bridge, not on `lo`) |

**Questions**

1. **Which transport protocol carries F1-U?** **UDP** (port 2152), with GTP-U on top, over IP (frames 37 and 38).
2. **What does a GTP-U TEID identify?** The tunnel endpoint on the **receiving** node, meaning which UE bearer (here DRB 1 of the UE) the packet belongs to. Each side allocates its own TEID and announces it over F1-C: the CU announced `7dc29161` (frame 27) and the DU announced `815b5aed` (frame 28). That is why uplink and downlink packets carry different TEIDs.
3. **What direction is `127.0.0.4 → 127.0.0.3`?** DU → CU, which is **uplink**. These packets carry the UE's echo requests (and the IPv6 Router Solicitations) towards the network.
4. **How would you distinguish F1-U from N3 in a capture?** Both use GTP-U on UDP 2152, so neither "GTP" nor the port is enough. Classify by **endpoints**: F1-U is DU ↔ CU (127.0.0.4 ↔ 127.0.0.3), N3 is CU ↔ UPF (192.168.70.129 ↔ 192.168.70.134). Two further hints: the F1-U payload is a PDCP PDU (with an NR RAN container extension on downlink), whereas N3 carries plain IP packets with a PDU Session Container; and F1-U TEIDs appear in F1AP UE Context Setup while N3 TEIDs appear in NGAP.
5. **Why do F1-C and F1-U use different transport protocols?** F1-C carries signaling, which must arrive reliably and in order and benefits from SCTP's multi-streaming and multi-homing. F1-U carries bulk, latency-sensitive user data. Retransmission is already handled by upper layers (RLC AM, PDCP, end-to-end TCP), so lightweight **UDP** avoids extra overhead and head-of-line blocking, while GTP-U adds per-bearer tunneling with TEIDs.


---

## 8. Task 6 — Compare monolithic and split gNB

| Feature | Lab 3: monolithic gNB | Lab 4: split CU/DU |
|---|---|---|
| Number of RAN processes | 1 (`nr-softmodem`) | 2 (gNB-CU + gNB-DU `nr-softmodem`) |
| Location of RRC/PDCP | Inside the single gNB | gNB-CU (127.0.0.3) |
| Location of RLC/MAC/PHY | Inside the single gNB | gNB-DU (127.0.0.4) |
| F1-C present? | No (internal function calls) | Yes: F1AP/SCTP 38472 (frames 1–32) |
| F1-U present? | No | Yes: GTP-U/UDP 2152 (63 T-PDUs) |
| UE connected to the same 5GC? | Yes | Yes: same OAI CN5G (AMF 192.168.70.132), reached through the CU |
| End-to-end ping works? | Yes | Yes: 10/10, 0 % loss, avg RTT 22.144 ms |
| SSB NR-ARFCN / UE `-C` | 641280 / 3619200000 | 630048 / 3450720000 |
| N2/N3 terminate at | The gNB | The gNB-CU (NG-U 192.168.70.129 ↔ UPF 192.168.70.134) |

**Explanation.** Splitting the gNB moves RRC and PDCP into the CU and keeps RLC, MAC and PHY in the DU, so traffic that used to pass between layers inside one process now crosses a standardized interface: F1-C (F1AP over SCTP) for signaling and F1-U (GTP-U over UDP) for user data. This split is invisible to the UE, because the air interface is still produced by the DU's PHY and MAC exactly as a monolithic gNB would produce it, and the RRC and NAS messages the UE exchanges are the same, only relayed through the DU inside F1AP RRC containers. The UE therefore still has to synchronize to the DU's SSB (which is why `-C` had to change to 3450720000), decode SIB1, perform random access, register with the AMF over NAS, and establish a PDU session before it gets an IP address. Our capture confirms this: F1 Setup completed at about 0.017 s, long before any UE existed, and the UE only became usable after the RRC, registration and UE Context Setup procedures that followed at about 21.45 s. In short, the split changes how the RAN is built internally, but not what the UE has to do to get service.

---

## 9. Troubleshooting

| Symptom | First checks |
|---|---|
| DU cannot establish F1 Setup | CU running; CU/DU F1 IP and ports; SCTP association; `f1ap` capture |
| UE repeatedly fails to decode SIB1 | Verify DU `absoluteFrequencySSB` and UE `-C`; check PHY/MAC synchronization and DU logs |
| UE syncs but Registration fails | Check CU/AMF logs, subscriber credentials (IMSI/key/OPc), RRC/NAS progress |
| UE attaches but no `oaitun_ue1` | Check PDU Session Establishment, 5GC SMF/UPF and UE logs |
| Ping fails | Confirm the tunnel, external DN reachability, routing, and user-plane path |
| Only F1AP appears in PCAP | Run Ping; verify PDU session; verify F1-U endpoints/port and capture interface |
| GTP appears but could be N3 | Check source/destination IPs against DU, CU and UPF addresses |
| Wireshark does not decode UDP 2152 as GTP-U | Examine UDP payload and optionally use **Analyze → Decode As…** |

**Do not** assume a screenshot of F1 Setup proves that UE Registration and end-to-end connectivity have succeeded. Validate each stage separately.

---

## 10. Submission checklist

Submit **one Markdown or PDF report** and **one PCAP file** containing:

- [x] CU/DU configuration summary and frequency check.
- [x] CU/DU and nrUE running evidence.
- [x] UE tunnel IP and successful Ping result (loss and RTT).
- [x] One PCAP containing **F1-C and F1-U**.
- [x] F1 Setup Request/Response screenshot.
- [x] Three or more UE-related F1AP messages with explanation.
- [x] At least one GTP-U T-PDU screenshot with TEID and endpoints.
- [x] Explanation distinguishing F1-C, F1-U and N3.
- [x] Completed monolithic-vs-split comparison table.
- [x] Answers to all analysis questions.

All screenshots, packet numbers, TEIDs and measurements must come from **your own experiment**.

## 11. Suggested grading rubric

| Category | Points |
|---|---:|
| Correct CU/DU deployment and parameter verification | 20 |
| F1-C capture and F1 Setup analysis | 20 |
| RRC/UE Context messages over F1-C | 20 |
| F1-U GTP-U and TEID analysis | 20 |
| UE tunnel and end-to-end Ping | 10 |
| Monolithic vs split comparison and report quality | 10 |
| **Total** | **100** |

---

## 12. Stop the experiment

Stop nrUE, DU and CU with `Ctrl+C`. Then stop CN5G:

```bash
cd ~/oai-cn5g
docker compose down
```

**Takeaway:** In the same OAI 5G SA network, **F1-C** exchanges RAN control signaling between CU and DU using **F1AP/SCTP**, while **F1-U** transports UE user-plane data between CU and DU using **GTP-U/UDP**. A successful UE Ping validates the end-to-end data path, while the PCAP shows what happens on the internal split-RAN interface.
