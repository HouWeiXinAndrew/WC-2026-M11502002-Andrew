# Lab 3: Deploying a 5G Standalone Network Using OpenAirInterface

## 1. Goal

In the previous labs, you analyzed 5G signaling messages and network performance using provided captures and measurements.

In this lab, you will deploy your own 5G Standalone (SA) network using OpenAirInterface (OAI), verify the complete connection, investigate RFsimulator communication, perform active network measurements, and troubleshoot a controlled connection failure.

Unlike the previous labs, you will collect measurements directly from your own running 5G network.

**Learning Objectives**

After completing this lab, you should be able to:

1. Deploy an OAI gNB, nrUE, and 5G Core Network.
2. Identify the main stages of 5G SA connection establishment.
3. Verify communication between gNB and nrUE through RFsimulator.
4. Measure user-plane connectivity under different packet sizes.
5. Diagnose connection failures using OAI and 5GC logs.
6. Explain the limitations of a software-simulated 5G network.

## 2. Experimental Environment

Use the *OAI v2.4.0 Build Guide* to prepare the environment before class.

| Component | Configuration |
|---|---|
| Ubuntu | 24.04 |
| OAI RAN | v2.4.0 |
| 5G Core | OAI CN5G |
| gNB | nr-softmodem |
| UE | nr-uesoftmodem |
| Radio interface | RFsimulator |
| Band | n78 |
| Subcarrier spacing | 30 kHz |
| Resource blocks | 106 |
| UE IMSI | 001010000000001 |
| External DN | 192.168.70.135 |

The experiment uses one Ubuntu machine without physical RF hardware.

## 3. Task 1 — Deploy and Verify the Complete 5G SA Network

Start the OAI CN5G:

```bash
cd ~/oai-cn5g
docker compose up -d
docker compose ps
```

Start the gNB and nrUE using the commands in the Build Guide.

Observe their terminal outputs and locate evidence of the following stages.

| Stage                        | Successful? | Evidence from logs                                                                                                                                                                             |
| ---------------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CN5G initialization          | YES         | `docker compose up -d` started 10/10 containers; `docker compose ps` shows all of them `Up (healthy)` (ims, mysql, oai-amf, oai-ausf, oai-ext-dn, oai-nrf, oai-smf, oai-udm, oai-udr, oai-upf) |
| gNB NG Setup                 | YES         | `[NGAP] Send NGSetupRequest to AMF` → `[NGAP] Received NGSetupResponse from AMF` (AMF OAI-AMF, PLMN 001/01, SST=0x01)                                                                          |
| UE synchronization           | YES         | `Initial sync successful, PCI: 0` and `UE synchronized! decoded_frame_rx=644 ...`, then `SIB1 decoded`                                                                                         |
| RRC connection establishment | YES         | `4-Step RA procedure succeeded`, `Received NR_RRCSetup`, `State = NR_RRC_CONNECTED`                                                                                                            |
| UE Registration              | YES         | `Received Registration Accept with result 3GPP`, then `Send ... message(RegistrationComplete)`                                                                                                 |
| PDU Session Establishment    | YES         | `Received PDU Session Establishment Accept, UE IPv4: 10.0.0.2`                                                                                                                                 |
| UE tunnel creation           | YES         | `[OIP] Interface oaitun_ue1 successfully configured, IPv4 10.0.0.2`                                                                                                                            |

Take screenshots showing the gNB and UE successfully connected.


| Stage                        | Evidence                                                                                                                                                                            |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CN5G initialization          | ![task1-CN5G initialization-docker compose up](https://hackmd.io/_uploads/ByWYmLWAfg.png) ![task1-CN5G initialization-docker compose ps](https://hackmd.io/_uploads/H1MsQUbAGe.png) |
| gNB NG Setup                 | ![task1-gNB NG Setup](https://hackmd.io/_uploads/rJH2m8-0Gg.png)                                                                                                                    |
| UE synchronization           | ![task1-UE synchronization](https://hackmd.io/_uploads/BJHam8-Cfx.png)                                                                                                              |
| RRC connection establishment | ![task1-RRC connection establishment](https://hackmd.io/_uploads/HJMRmIbRzx.png)                                                                                                    |
| UE Registration              | ![task1-UE Registration](https://hackmd.io/_uploads/SkKyVIbAfg.png)                                                                                                                 |
| PDU Session Establishment    | ![task1-PDU Session Establishment](https://hackmd.io/_uploads/HyJW4U-Cfe.png)                                                                                                       |
| UE tunnel creation           | ![task1-UE tunnel creation](https://hackmd.io/_uploads/HJC-VIb0Mg.png)                                                                                                              |

**Questions:**

**1. Does a successful NG Setup guarantee that a UE can register? Explain.**

No. NG Setup only establishes the N2 (NGAP over SCTP) relationship between the gNB and the AMF: the gNB announces its identity, PLMN and supported slices, and the AMF accepts it. Registration is a separate per-UE procedure that also needs the UE's identity to be known in the subscriber database, authentication to succeed in the AUSF/UDM, and a valid PLMN and slice match. <mark>In my Task 5 experiment NG Setup was still successful and the AMF still listed the gNB as "Connected", yet the UE with IMSI 001010000000099 was rejected (see Task 5 evidence).</mark>

**2. Why can the UE establish an RRC connection but still fail Registration?**

RRC connection setup is an access-stratum procedure between the UE and the gNB (random access, `RRCSetup`, `RRCSetupComplete`). It does not check who the subscriber is. Registration is a NAS procedure between the UE and the AMF, which involves the core network and authenticates the subscriber. <mark>In my failed run the UE reached `NR_RRC_CONNECTED` and sent a Registration Request, but then received `FGS_REGISTRATION_REJECT` (see Task 5 evidence).</mark>

**3. What additional procedure is required after Registration before user data can be exchanged?**

A PDU Session Establishment is required. After `RegistrationComplete` the UE sends a `PduSessionEstablishRequest`; the SMF and UPF set up the session, the gNB sets up a data radio bearer (DRB 1), and the UE receives an IP address. <mark>In my log this appears as `Added drb 1`, `PDU Session Establishment Accept, UE IPv4: 10.0.0.2` and the `oaitun_ue1` interface (Task 1 evidence above).</mark>

## 4. Task 2 — Investigate RFsimulator Communication

OAI RFsimulator replaces the physical RF interface with software-based communication.

The gNB and nrUE exchange IQ samples through a TCP connection.

Check the running processes:

```bash
pgrep -a nr-softmodem
pgrep -a nr-uesoftmodem
```

Inspect the TCP connections:

```bash
sudo ss -tnp | grep 4043
```

If necessary, inspect the listener:

```bash
sudo ss -ltnp | grep 4043
```

The default RFsimulator port in this experiment is 4043.

Complete the table:

| Item                           | Observed Value                                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------------------------------ |
| gNB process ID                 | 11520 (`nr-softmodem`; 11521 is the parent `sudo` process)                                             |
| nrUE process ID                | 21465 (`nr-uesoftmodem`; 21466 is the parent `sudo` process)                                           |
| RFsimulator transport protocol | TCP                                                                                                    |
| RFsimulator port               | 4043                                                                                                   |
| TCP connection state           | ESTAB (established); gNB socket in LISTEN                                                              |
| Connected process(es)          | `nr-uesoftmodem` (pid 21465, client, local port 40348) ↔ `nr-softmodem` (pid 11520, server, port 4043) |

Take one screenshot showing the TCP connection.
![task2](https://hackmd.io/_uploads/ry13NLWRzx.png)

<mark>The screenshot shows `LISTEN *:4043` owned by `nr-softmodem` (pid 11520) and an `ESTAB` connection `127.0.0.1:40348 → 127.0.0.1:4043` owned by `nr-uesoftmodem` (pid 21465). The gNB is the server and the UE is the client, both on the same machine.</mark>

**Questions:**

**1. Why does RFsimulator use a TCP connection instead of a physical antenna?**

RFsimulator replaces the RF front end with software. The gNB and UE are normal processes on one computer, so the digital IQ samples that would have gone to a radio are sent over a local TCP socket. TCP also gives a reliable, ordered stream, which makes testing the protocol stack repeatable without any radio hardware.

**2. Are the TCP packets exchanged by RFsimulator the same as the UE's application-layer IP packets?**

No. The RFsimulator TCP stream carries IQ samples, the simulated radio waveform that contains PBCH, PDCCH, PDSCH, PUSCH and so on. The UE's application packets (for example the ICMP ping on `oaitun_ue1`) are the payload that is encoded inside that waveform. They are carried through PDCP/RLC/MAC/PHY, the GTP-U tunnel and the UPF, and are completely separate from the TCP connection between the two softmodem processes.

**3. Does using RFsimulator mean that the UE and gNB are performing no physical-layer processing? Explain.**

No. Both sides still run the full NR PHY: <mark>the UE log shows PSS/SSB search, PBCH decoding, frame synchronization and PRACH, and the gNB log shows frame parameters, OFDM symbol size 2048 and TDD slot configuration (Task 1 screenshots)</mark>. Only the analogue radio and the over-the-air channel are replaced by the TCP link.

## 5. Task 3 — Verify End-to-End Connectivity

Check the UE tunnel interface:

```bash
ip -4 addr show oaitun_ue1
```

Test connectivity to the External DN:

```bash
ping -I oaitun_ue1 -c 10 192.168.70.135
```

Record:

| Measurement         | Observed Value |
| ------------------- | -------------- |
| UE IP address       | 10.0.0.2       |
| Destination IP      | 192.168.70.135 |
| Packets transmitted | 10             |
| Packets received    | 10             |
| Packet loss (%)     | 0%             |
| Minimum RTT (ms)    | 20.781         |
| Average RTT (ms)    | 28.181         |
| Maximum RTT (ms)    | 35.741         |

Take a screenshot of the Ping result.
![task3](https://hackmd.io/_uploads/r1LkUUbCGx.png)

<mark>The screenshot shows `oaitun_ue1` with `10.0.0.2/24`, ping sent `from 10.0.0.2 oaitun_ue1`, 10/10 received, 0% loss, `rtt min/avg/max/mdev = 20.781/28.181/35.741/5.118 ms`, TTL 63.</mark>

**Questions:**

**1. Why must Ping use the `oaitun_ue1` interface in this experiment?**

The UE tunnel `oaitun_ue1` is the Linux interface that represents the 5G data path: packets sent into it are carried by the nrUE through the simulated radio, the gNB, GTP-U and the UPF. Without `-I oaitun_ue1`, the ping would use the machine's normal default route (for example Docker or Ethernet) and reach 192.168.70.135 without going through the 5G network at all, which would prove nothing about the 5G connection.

**2. Does successful Registration guarantee successful user-plane connectivity?**

No. Registration only establishes the UE's control-plane (NAS) context with the AMF. User data needs a PDU session (SMF/UPF, GTP-U tunnel, DRB) and working routing to the data network. A failure in any of those (UPF, SMF, DNN or slice mismatch, routing or NAT) would still leave the UE registered but unable to ping. <mark>Here the session and tunnel were established, so the ping worked (screenshot above).</mark>

**3. What does a successful Ping prove, and what does it not prove about the simulated 5G system?**

It proves the whole chain works end to end: UE tunnel → NR PHY/MAC/RLC/PDCP in the simulated link → gNB → GTP-U → UPF → external DN, and back. It does not prove real radio performance: there is no antenna, propagation, fading, interference or hardware impairment, so the RTT and loss say nothing about over-the-air quality. It also doesn't show throughput capacity or behaviour with many UEs.

## 6. Task 4 — Investigate the Effect of Packet Size

You will now investigate whether changing the ICMP packet size affects the observed RTT.

Perform three experiments using the same established 5G connection.

**Scenario A — Small Packets**

```bash
ping -I oaitun_ue1 -s 56 -c 30 -i 0.2 192.168.70.135
```

**Scenario B — Medium Packets**

```bash
ping -I oaitun_ue1 -s 512 -c 30 -i 0.2 192.168.70.135
```

**Scenario C — Large Packets**

```bash
ping -I oaitun_ue1 -s 1200 -c 30 -i 0.2 192.168.70.135
```

Note: `-s` specifies ICMP payload size, not the complete IP packet size.

Complete the table:

| Scenario | ICMP Payload | Avg. RTT (ms) | Max. RTT (ms) | Packet Loss (%) |
| -------- | ------------:| -------------:| -------------:| ---------------:|
| A        |     56 bytes |        25.583 |        29.111 |              0% |
| B        |    512 bytes |        27.909 |        38.266 |              0% |
| C        |   1200 bytes |        30.930 |        40.978 |              0% |

(Corresponding IP packet sizes: 84, 540 and 1228 bytes, as printed by ping.)

Calculate the RTT difference:

```text
RTT Difference = Average RTT(C) - Average RTT(A)
               = 30.930 - 25.583
               = 5.347 ms
```

**Questions:**

**1. Which scenario has the highest average RTT?**

<mark>Scenario C (1200-byte payload) has the highest average RTT at 30.930 ms, compared with 27.909 ms for B and 25.583 ms for A.</mark>

**2. Does increasing packet size always increase RTT in your measurements?**

In my three runs the average RTT did increase with size (25.583 → 27.909 → 30.930 ms), but I would not claim that it always does. <mark>The maximum RTT for B (38.266 ms) is much higher than its average, and the standard deviation grows from 1.748 ms (A) to 3.778 ms (B) and 3.142 ms (C), so some of the difference is jitter.</mark> My Task 3 baseline with the same 56-byte payload averaged 28.181 ms against 25.583 ms in Scenario A, a 2.6 ms difference from repeating almost the same test. That is about as large as the A→B step (2.3 ms), so the A→B increase in particular cannot be attributed to packet size with confidence.

**3. Why might a larger packet require more transmission or processing resources?**

A larger packet has more bits to encode, modulate and transmit, so the MAC scheduler must allocate more resource blocks or symbols and possibly more slots. Each layer (PDCP, RLC, MAC, PHY) also processes more bytes (ciphering, integrity, segmentation, coding, copying), and the GTP-U tunnel and UPF forward more data in both directions. The 1228-byte IP packet is below the 1500-byte MTU, so IP fragmentation did not occur.

**4. What factors other than packet size could affect RTT in a software-based 5G network?**

Everything runs on one machine, so CPU load and thread scheduling directly affect timing. Other factors are the TDD slot pattern (a ping has to wait for UL and DL slots), the RFsimulator TCP buffering, Docker networking and the UPF, and background processes. The run-to-run difference between Task 3 and Scenario A shows this variation.

**5. Can these results be interpreted as real over-the-air 5G latency? Explain.**

No. There is no real radio channel: no propagation delay, fading, interference, HARQ retransmissions caused by radio errors, or RF hardware delay. The RTT here is dominated by software processing, the slot structure and CPU scheduling on one machine, and 0% loss shows an ideal channel. Real 5G latency would be different and could not be derived from these numbers.

Do not assume that RTT must increase with packet size. Use your measured values to support your explanation.


| Scenario | Evidence                                                       |
| -------- | -------------------------------------------------------------- |
| A        | ![task4-scenario-a](https://hackmd.io/_uploads/rJWevLWRzg.png) |
| B        | ![task4-scenario-b](https://hackmd.io/_uploads/rJngP8ZCMe.png) |
| C        | ![task4-scenario-c](https://hackmd.io/_uploads/ByPbvLZ0Mx.png) |


## 7. Task 5 — Diagnose a Controlled UE Registration Failure

In this task, you will deliberately introduce one configuration error and investigate why the UE cannot complete registration.

**Important:** Save the original successful UE command before starting. Do not modify the CN5G subscriber database.

### Step 1 — Stop the UE

Press `Ctrl+C` in the nrUE terminal. Keep the CN5G and gNB running.

### Step 2 — Launch the UE with an Incorrect IMSI

Change only the IMSI:

```bash
sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3619200000 \
  --uicc0.imsi 001010000000099 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1
```

The IMSI `001010000000099` is not provisioned in the standard classroom subscriber configuration. Verify this assumption against the instructor's database before class.

Observe the UE and AMF logs.

```bash
cd ~/oai-cn5g
docker compose logs --since=2m oai-amf
```

Allow time for registration attempts, then stop the UE.

Complete the table:

| Item                        | Observation                                                                                                                                                                                                                                                                                                                                                                               |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Modified parameter          | `--uicc0.imsi` changed from 001010000000001 to 001010000000099; all other parameters unchanged                                                                                                                                                                                                                                                                                            |
| UE synchronized with gNB?   | Yes - `UE synchronized! decoded_frame_rx=98 ...`, `SIB1 decoded`                                                                                                                                                                                                                                                                                                                          |
| RRC connection established? | Yes — `4-Step RA procedure succeeded`, `Received NR_RRCSetup`, `State = NR_RRC_CONNECTED`                                                                                                                                                                                                                                                                                                 |
| Registration successful?    | No — UE received `FGS_REGISTRATION_REJECT with length 4`; no Registration Accept, no PDU session, no `oaitun_ue1`                                                                                                                                                                                                                                                                         |
| Error or rejection evidence | UE: `Received NAS_DOWNLINK_DATA_IND type FGS_REGISTRATION_REJECT`. AMF: `Received IMSI 001010000000099`; AUSF response `404 ... "cause":"CONTEXT_NOT_FOUND","detail":"Resource corresponding to User imsi-001010000000099 not found"`; `[error] Request Authentication Vectors failure`; `5GMM state transition: COMM-PROC-INIT -> 5GMM-DEREGISTERED (event: [REGISTRATION_REJECT_SENT])` |
| Likely cause                | The IMSI 001010000000099 is not provisioned in the 5GC subscriber database, so the AUSF could not find the subscriber and the AMF could not obtain authentication vectors.                                                                                                                                                                                                                |

If the expected registration rejection does not appear, report the actual logs instead of inventing a failure message.



| EVIDENCE — Failed connection: UE log                               | EVIDENCE — Failed connection: AMF log                               |
| ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| ![task5-ue-log-failure](https://hackmd.io/_uploads/r1fLvObRMe.png) | ![task5-amf-log-failure](https://hackmd.io/_uploads/HkMODuWRzg.png) |

(The `404` strings in `02:02:42.404`, `02:44:30.404` etc. are only millisecond digits in heartbeat timestamps matched by the grep pattern and are unrelated to the failure. The failure lines are the ones at `02:42:36`.)

### Step 3 — Restore the Original IMSI

Restart the nrUE using:

```text
--uicc0.imsi 001010000000001
```

Confirm that the UE successfully registers again and that Ping works.

**Questions:**

**1. Why can the UE potentially synchronize with the gNB even when the IMSI is incorrect?**

Synchronization (PSS/SSS/PBCH, SIB1) and random access/RRC setup happen on the radio side and use only the cell's broadcast signals and the gNB's RRC procedures. The IMSI is part of the UE's NAS identity and is first sent inside the Registration Request, after the RRC connection exists. <mark>My UE log shows the correct sync and RRC connection even with IMSI ...099, and the failure only appears at the NAS Registration step.</mark>

**2. Which network function is involved in UE authentication?**

The AUSF (Authentication Server Function) performs the authentication, working with the UDM and UDR, which hold the subscriber credentials. The AMF triggers it by requesting authentication vectors from the AUSF. <mark>In my AMF log, the AMF sends the request to `http://oai-ausf:8080/nausf-auth/v1/ue-authentications` and the AUSF answers `404 CONTEXT_NOT_FOUND` for the unknown IMSI.</mark>

**3. Why does changing an IMSI affect Registration but not the RFsimulator TCP connection itself?**

The RFsimulator TCP connection is just a transport for IQ samples between the two processes, and it is set up from the server address and port (127.0.0.1:4043) before any subscriber information is exchanged. The IMSI only matters in the NAS layer, where the core network looks it up. <mark>My failed run shows `Connection to 127.0.0.1:4043 established` and a successful sync with the wrong IMSI, so the link was fine and only Registration failed.</mark>

**4. What evidence proves that restoring the correct configuration solved the problem?**

<mark>Only the IMSI was changed back, and the results change at the Registration step: the failed run ended with `FGS_REGISTRATION_REJECT` and AMF/AUSF `404 CONTEXT_NOT_FOUND`, while the recovered run shows `Registration Accept`, `PDU Session Establishment Accept`, `oaitun_ue1` configured, a working ping with 0% loss, and the AMF table showing IMSI ...001 as `5GMM-REGISTERED`.</mark> The recovered registration is a new one, not a leftover: the AMF UE NGAP ID is now 0x03 and the GUTI is different from the earlier session. The synchronization and RRC results were identical in both runs, which shows that the IMSI was the only cause.

Submit evidence from both the failed and recovered connections.



| EVIDENCE — Recovered connection: UE log                             | EVIDENCE — Recovered connection: ping and AMF state                  |
| ------------------------------------------------------------------- | -------------------------------------------------------------------- |
| ![task5-ue-log-recovery](https://hackmd.io/_uploads/rkTfjuW0fg.png) | ![task5-ping-and-amf-log](https://hackmd.io/_uploads/SJTXodZ0Me.png) |

<mark>After recovery the UE received `Registration Accept` and `PDU Session Establishment Accept, UE IPv4: 10.0.0.2`, `oaitun_ue1` was configured, ping gave 10/10 received, 0% loss (min/avg/max 11.439/20.601/25.930 ms), and the AMF table shows IMSI 001010000000001 as `5GMM-REGISTERED` with a GUTI, while IMSI 001010000000099 remains `5GMM-DEREGISTERED` with no GUTI.</mark>

## 8. Task 7 — Explain the Relationship Between Simulation and Real 5G

Your OAI gNB and nrUE perform 5G NR protocol processing even though they do not use physical antennas.

Complete the comparison:

| Feature                      | OAI RFsimulator                                                                     | Physical 5G System                                                               |
| ---------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| gNB and UE protocol stack    | Full OAI NR stack runs (PHY, MAC, RLC, PDCP, RRC, NAS; NGAP/GTP-U towards the core) | Same protocol stack, running on real gNB and UE devices                          |
| OFDM signal processing       | Done in software (IQ samples generated and decoded), then passed over TCP           | Done in baseband hardware/software, then converted by DAC/ADC to RF              |
| Physical antennas            | None                                                                                | Required (gNB and UE antennas)                                                   |
| RF hardware impairments      | Not present (ideal samples)                                                         | Present: noise, non-linearity, frequency offset, phase noise, calibration errors |
| Wireless propagation effects | Not present (no path loss, fading, interference)                                    | Present: path loss, multipath fading, shadowing, interference, mobility          |
| 5G Core connectivity         | Real OAI CN5G (AMF, SMF, UPF, ...) via NGAP and GTP-U                               | Same, over a real backhaul network                                               |

**Questions:**

**1. Which parts of the 5G system remain functional when physical RF hardware is replaced with RFsimulator?**

Almost everything above the antenna: the PHY processing, MAC/RLC/PDCP/RRC, NAS signalling, NG Setup, authentication, registration, PDU sessions, GTP-U and the 5G Core. My experiment shows this: synchronization, random access, registration, PDU session and ping all worked without any RF hardware.

**2. Why is RFsimulator useful for testing the 5G protocol stack?**

It needs only one computer, so there is no cost, licensing, shielding or radio interference to manage. Tests are repeatable and controlled, and it is easy to inject configuration errors (such as my wrong-IMSI test) and read the logs of every layer. It is good for development, debugging and learning before moving to real hardware.

**3. Why might latency and throughput differ between RFsimulator and a real 5G deployment?**

RFsimulator timing depends on CPU scheduling and TCP processing on one machine, and the channel is ideal, so there are no radio errors or retransmissions. A real deployment has propagation, fading, interference, HARQ retransmissions, adaptive modulation based on channel quality, RF hardware delay and real backhaul. Throughput in the simulator is limited by CPU and software, not by spectrum or radio conditions.

**4. What would need to change if you wanted to test a physical UE instead of OAI nrUE?**

The gNB must use a real RF front end (for example a USRP) instead of `--rfsim`, with antennas and legal or shielded transmission on the chosen band. The gNB's band, frequency (SSB/ARFCN), bandwidth, SCS and PLMN settings must match what the commercial UE supports. The UE needs a SIM or eSIM whose IMSI, Ki and OPc are provisioned in the core's subscriber database, and the nrUE is no longer started at all.

## 9. Final Analysis

Answer each question in three to five sentences.

**1. Describe the complete process from starting the OAI 5GC to successful user-plane connectivity.**

First I started the core with `docker compose up -d`, and `docker compose ps` showed all 10 containers healthy. Then I started the gNB, which sent an NGSetupRequest and got an NGSetupResponse from the AMF, and then the nrUE, which connected to the gNB over RFsimulator on TCP port 4043. The UE synchronized to the cell (PBCH/SIB1), completed random access and RRC setup, and then sent a Registration Request. After authentication and security mode it received Registration Accept, and then the PDU session was established, which gave it IP 10.0.0.2 and created `oaitun_ue1`. Finally the ping through `oaitun_ue1` to 192.168.70.135 succeeded with 0% loss. <mark>(Evidence: Task 1 screenshots, Task 3 screenshot.)</mark>

**2. Explain the difference between an RFsimulator TCP connection and a 5G PDU Session.**

The RFsimulator TCP connection is a lower-layer transport between the nr-softmodem and nr-uesoftmodem processes: it replaces the radio and carries IQ samples on port 4043. A PDU session is a 5G core-network logical connection between the UE and the data network, set up by the SMF and UPF, with a GTP-U tunnel, a DRB on the radio side and an IP address for the UE. The TCP connection existed before any registration and was unaffected by the wrong IMSI, while the PDU session only exists after successful Registration. User application packets such as ping travel inside the PDU session, which is itself carried inside the IQ samples sent over TCP. <mark>(Evidence: Task 2 screenshot, Task 1 PDU session and tunnel screenshots.)</mark>

**3. Based on your measurements, how did packet size affect RTT and packet loss?**

Average RTT rose from 25.583 ms (56 bytes) to 27.909 ms (512 bytes) and 30.930 ms (1200 bytes), so the 1200-byte payload was 5.347 ms slower than the 56-byte one. Packet loss was 0% for all three sizes, so size did not cause loss in this ideal simulated channel. However, the Task 3 baseline with the same 56-byte payload averaged 28.181 ms, 2.6 ms away from Scenario A, so part of the differences is run-to-run variation (jitter and CPU scheduling). I conclude that larger packets tended to take longer in my runs, but I would not claim a precise linear effect from three single runs. <mark>(Evidence: Task 4 screenshots.)</mark>

**4. Describe the registration failure you introduced and explain how you diagnosed and recovered from it.**

I changed only the UE IMSI to 001010000000099, which is not provisioned in the subscriber database. The UE still synchronized and entered `NR_RRC_CONNECTED`, but received `FGS_REGISTRATION_REJECT`. In the AMF log I found the cause: the AMF asked the AUSF for authentication vectors and the AUSF returned `404 CONTEXT_NOT_FOUND` for `imsi-001010000000099`, followed by `Request Authentication Vectors failure` and `REGISTRATION_REJECT_SENT`. I recovered by restarting the nrUE with the original IMSI 001010000000001, after which Registration Accept, the PDU session, `oaitun_ue1` and a 0%-loss ping returned, and the AMF showed the UE as `5GMM-REGISTERED`. <mark>(Evidence: Task 5 failed and recovered screenshots.)</mark>

**5. Identify two limitations of this simulated environment and explain how they might affect experimental results.**

First, there is no real radio channel: no path loss, fading, interference, mobility or RF impairment, so loss was always 0% and the RTT does not represent over-the-air latency or coverage. Results about reliability or radio performance would be too optimistic. Second, everything runs on one machine, so the gNB, UE, core containers and my measurement tools share the CPU. This adds jitter and timing noise (for example the 2.6 ms difference between two 56-byte tests), can limit throughput, and means timing results depend on the hardware and load of the host, not on 5G itself.

## 10. Submission Checklist

Submit one Markdown or PDF report containing:

- Completed 5G SA connection verification table.
- gNB and nrUE connection screenshots.
- RFsimulator TCP connection evidence.
- Baseline Ping measurement.
- Three packet-size comparison results.
- RTT difference calculation.
- Controlled failure and recovery evidence.
- RFsimulator versus physical 5G comparison.
- Answers to the final analysis questions.

All measurements and log evidence must come from your own experimental environment.

## 11. Grading Rubric

| Item | Points |
|---|---:|
| Complete 5G SA deployment | 15 |
| RFsimulator TCP connection analysis | 15 |
| End-to-end user-plane verification | 15 |
| Packet-size experiment and analysis | 20 |
| Controlled failure and recovery | 20 |
| Simulation limitations and final analysis | 15 |
| **Total** | **100** |

Vote TOP3：https://forms.gle/Mvza68DFBzPcZXfW6
