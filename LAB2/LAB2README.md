# Lab 2: gNB TDD Traffic Analysis
Also available at my [HackMD](https://hackmd.io/@andrewbartholomeo/HkH0Dh9Tze)
## 1. Goal

In this lab, you will compare three 5-second UDP iperf3 tests. You will learn how TDD slot allocation can affect downlink and uplink performance.

You will also read the broadcast TDD configuration from SIB1 in Wireshark, draw the downlink/uplink time allocation, and relate the configuration to the iperf3 results.

You do not need to understand every packet. Follow the steps, complete the tables, and answer the short questions.

## 2. Test Scenarios and Files

| Scenario | TDD setting | Traffic direction | Main idea |
| --- | --- | --- | --- |
| A | DL-heavy | Downlink | The TDD setting favors the traffic direction |
| B | DL-heavy | Uplink | There are fewer opportunities for uplink traffic |
| C | Balanced | Uplink | More uplink opportunities are available than in B |

[Download the lab files from Google Drive](https://drive.google.com/drive/u/0/folders/1gTgosNkWFuak_sXbHFTxqaRqXb15Wz9T).

### Two Types of Packet Captures

Two types of packet captures are provided in this lab:

- **Configuration captures:** Recorded separately during initial cell access to show the broadcast SIB1 messages for the two TDD settings. Use these files to read and draw the common TDD configuration. Only two configuration captures are required because Scenarios A and B share the DL-heavy setting.
- **Scenario captures:** Recorded during the iperf3 tests. Use these files to analyze user-plane packets and the TCP control connection. Separate captures are provided for A, B, and C because each scenario uses a different combination of TDD setting and traffic direction.

The configuration captures are reference recordings for the corresponding settings, not simultaneous recordings of the original iperf3 runs. The scenario-to-configuration mapping below is supplied by the instructor; it cannot be established from the iperf3 JSON alone. Configuration captures are not used to calculate test throughput, loss, jitter, RTT, or packet intervals.

### Configuration Captures — Task 4 Only

| TDD setting | Applies to | Configuration capture |
| --- | --- | --- |
| DL-heavy | Scenarios A and B | `tdd-dl-heavy-sib1.pcapng` |
| Balanced | Scenario C | `tdd-balanced-sib1.pcapng` |

### Performance Files — Tasks 1–3 and 5–8

| Scenario | iperf3 JSON | Scenario capture |
| --- | --- | --- |
| A | `scenario-a-downlink.json` | `scenario-a-dl-heavy-downlink.pcapng` |
| B | `scenario-b-uplink.json` | `scenario-b-dl-heavy-uplink.pcapng` |
| C | `scenario-c-uplink.json` | `scenario-c-balanced-uplink.pcapng` |

> The JSON files report the traffic direction and performance results. They do not contain the gNB TDD configuration. Use the configuration captures for SIB1 and the scenario captures for traffic analysis.

## 3. Task 1 — Confirm the Traffic Direction

Open each JSON file and find:

```text
start → test_start → reverse
```

- `reverse: 1` means downlink in this lab.
- `reverse: 0` means uplink in this lab.

Complete the table:

| Scenario | Reverse value | Direction |
| -------- | -------------:| --------- |
| A        |             1 | downlink  |
| B        |             0 | uplink    |
| C        |             0 | uplink    |

**Question:** Does the traffic direction shown in the JSON match the scenario description? Show the proof.

**Answer:** Yes, the traffic direction shown in the JSON matches the scenario description. The proof is:
| Scenario | Reverse value Direction | Proof                                                          |
| -------- | -----------------------:| -------------------------------------------------------------- |
| A        |              1 downlink | ![task1-scenario-a](https://hackmd.io/_uploads/B19won9pzl.png) |
| B        |                0 uplink | ![task1-scenario-b](https://hackmd.io/_uploads/HkvFi2qTMg.png) |
| C        |                0 uplink | ![task1-scenario-c](https://hackmd.io/_uploads/r1Osin9pMg.png) |


## 4. Task 2 — Compare Throughput

In each JSON file, find:

```text
end → sum → bits_per_second
```

Convert the value to Mbps:

```text
Throughput (Mbps) = bits_per_second ÷ 1,000,000
```

Complete the table:

| Scenario | Direction | Throughput (Mbps) |
| -------- | --------- | -----------------:|
| A        | Downlink  |  90.6541958358403 |
| B        | Uplink    |  109.11724969037475 |
| C        | Uplink    | 124.41674913023271 |

Calculate how much C improves over B:

```text
Improvement (%) = (C - B) ÷ B × 100
```

Answer:

1. What is the downlink throughput in Scenario A?
90.6541958358403 = 90.65 Mbps
2. What is the uplink throughput in Scenario B?
109.11724969037475 = 109.12 Mbps
3. Is the uplink throughput higher in B or C?
C is higher: 124.41674913023271 = 124.42 Mbps, compared with 109.12 Mbps in B.
4. By approximately what percentage does C improve over B?
Improvement (%) = (124.41674913 − 109.11724969) ÷ 109.11724969 × 100 = 15.2995 ÷ 109.1172 × 100 = 14.02115566811957 ≈ 14.02% (about +15.30 Mbps).

> Use the JSON `bits_per_second` value. Do not recalculate throughput using the other time fields.

Proof = 
| Scenario | Direction Throughput (Mbps) |                                                          Proof |
| -------- | --------------------------- | --------------------------------------------------------------:|
| A        | Downlink 90.6541958358403   | ![task2-scenario-a](https://hackmd.io/_uploads/B1-Aj39aze.png) |
| B        | Uplink 109.11724969037475   | ![task2-scenario-b](https://hackmd.io/_uploads/rygxh356Ml.png) |
| C        | Uplink 124.41674913023271   | ![task2-scenario-c](https://hackmd.io/_uploads/SkFZ33cTfx.png) |


## 5. Task 3 — Check the Trade-off

In each JSON file, find:

```text
end → sum → lost_percent
end → sum → jitter_ms
```

Complete the table:

| Scenario |  Throughput (Mbps) |     Packet loss (%) |         Jitter (ms) |
| -------- | ------------------:| -------------------:| -------------------:|
| B        | 109.11724969037475 | 0.23565878306653645 | 0.11619665995310689 |
| C        | 124.41674913023271 |   2.133919261135111 | 0.16204804777679407 |

Answer:

1. Does C have higher uplink throughput than B?
Yes, Scenario C has higher throughput (124.42 Mbps) than Scenario B (109.12 Mbps).
2. Does C have lower packet loss than B?
No, Scenario C has higher packet loss (2.13%) than Scenario B (0.24%), not lower.
3. What trade-off, if any, is visible when changing from B to C?
Transitioning from B to C demonstrates a performance trade-off where higher throughput is gained at the expense of degraded transmission quality, characterized by increased packet loss (rising from 0.24% to 2.13%) and higher jitter (rising from 0.12 ms to 0.16 ms).

A simple answer format is:

> Compared with Scenario B, Scenario C has ______ throughput, ______ packet loss, and ______ jitter. These results suggest ______.

Use the measured values. If one result improves while another worsens, explain that trade-off. Do not assume that increasing UL resources always increases packet loss.

Compared with Scenario B, Scenario C has higher throughput, higher packet loss, and higher jitter. These results suggest a trade-off: in this measurement the higher throughput came together with greater packet loss (about 9 times higher) and more jitter. This does not mean that adding UL resources always increases packet loss; the loss may also depend on how much traffic was offered, scheduling and radio conditions.

| Scenario | Proof                                                          |
| -------- | -------------------------------------------------------------- |
| B        | ![task3-scenario-b](https://hackmd.io/_uploads/SknQ33qpfx.png) |
| C        | ![task3-scenario-c](https://hackmd.io/_uploads/HyfBhhqazg.png) |


## 6. Task 4 — Read SIB1 and Draw the TDD Configuration

Use these **two configuration captures** for this task:

- `tdd-dl-heavy-sib1.pcapng`: Reference configuration for Scenarios A and B.
- `tdd-balanced-sib1.pcapng`: Reference configuration for Scenario C.

These files were recorded separately from the original iperf3 tests. Use them to examine the supplied reference settings.

### 4.1 What Is SIB1?

A gNB broadcasts system information so that a UE can learn how to access the cell.

- **MIB (Master Information Block):** Provides basic information, including parameters needed to find SIB1.
- **SIB1 (System Information Block Type 1):** Provides cell-access and common radio configuration information. In this TDD lab, it includes `tdd-UL-DL-ConfigurationCommon`.

You will read the **common TDD configuration in SIB1**, not decode every MIB/SIB field. This configuration describes DL, UL, and flexible time resources; it is not a record of the actual data scheduled in every slot.

### 4.2 Enable MAC-NR Decoding and Find SIB1

These captures contain NR MAC messages carried over UDP. **Enable MAC-NR-over-UDP decoding before applying the SIB1 filter.** Without this decoder, the packets may appear only as UDP and the SIB1 filter may return no results.

1. Open `tdd-dl-heavy-sib1.pcapng` in Wireshark.
2. Open **Analyze → Enabled Protocols**. Locate **MAC-NR over UDP** (`mac_nr_udp`) in the heuristic dissectors list and enable it. Depending on the Wireshark version, the heuristic list may be shown separately. Apply the change and reload the capture if needed.
3. Apply this display filter:

```text
   nr-rrc.systemInformationBlockType1_element
```

4. Select a SIB1 packet. In the packet details, expand:

```text
   NR Radio Resource Control (RRC) protocol
     → BCCH-DL-SCH-Message
       → message
         → c1
           → systemInformationBlockType1
             → servingCellConfigCommon
               → tdd-UL-DL-ConfigurationCommon
```

5. Read `referenceSubcarrierSpacing`, then expand `pattern1` and record:

```text
   dl-UL-TransmissionPeriodicity
   nrofDownlinkSlots
   nrofDownlinkSymbols
   nrofUplinkSlots
   nrofUplinkSymbols
```

6. Check whether `pattern2` is also present.
7. Repeat for `tdd-balanced-sib1.pcapng`.

Record the **decoded labels and units** shown in Wireshark. For example, record `30 kHz` or `5 ms`, not the enumeration number displayed in parentheses.

You can also locate packets containing the TDD configuration with:

```text
nr-rrc.tdd_UL_DL_ConfigurationCommon_element
```

Confirm that the selected TDD fields are inside **SIB1**. If a filter is not recognized in your version, try `nr-rrc` and locate `systemInformationBlockType1` manually.

> **Troubleshooting:** If packets still appear only as UDP, check that the **MAC-NR over UDP heuristic dissector** is enabled, rather than only the MAC-NR protocol. If the file still does not decode, ask the instructor to check the settings. Do not infer TDD parameters from packet timing. Graphical menu labels may vary by version.

### 4.3 Record the TDD Parameters

Complete the table for `pattern1`:

| Item                                 | DL-heavy: A/B            | Balanced: C              |
| ------------------------------------ | ------------------------ | ------------------------ |
| Configuration capture filename       | tdd-dl-heavy-sib1.pcapng | tdd-balanced-sib1.pcapng |
| SIB1 packet number                   | 2                        | 2                        |
| `referenceSubcarrierSpacing` (kHz)   | 30                       | 30                       |
| `dl-UL-TransmissionPeriodicity` (ms) | 5                        | 2.5                      |
| `nrofDownlinkSlots`                  | 7                        | 2                        |
| `nrofDownlinkSymbols`                | 6                        | 6                        |
| `nrofUplinkSlots`                    | 2                        | 2                        |
| `nrofUplinkSymbols`                  | 4                        | 4                        |
| Is `pattern2` present?               | no                       | no                       |

If `pattern2` is present, add another table containing its period and four slot/symbol counts. It uses the same reference SCS.

Take **one screenshot per configuration** showing the selected SIB1 packet and the expanded TDD fields. Include `pattern2` in an additional screenshot if it is present and does not fit.


| Configuration Capture Filename | Pattern1                                                              |
| ------------------------------ | --------------------------------------------------------------------- |
| tdd-dl-heavy-sib1.pcapng       | ![task4-heavy-pattern1](https://hackmd.io/_uploads/r1GK2296Gx.png)    |
| tdd-balanced-sib1.pcapng       | ![task4-balanced-pattern1](https://hackmd.io/_uploads/BJS5hnqafe.png) |

### 4.4 Draw the TDD Time Allocation

Draw one diagram for DL-heavy and one for balanced. **Use the same 0–5 ms time axis for both diagrams.** Derive each repeating period from SIB1 and repeat the shorter pattern as needed to fill the 5 ms interval. Mark every repeating-period boundary.

A spreadsheet, a drawing tool, or a hand-drawn diagram is acceptable. Include the diagrams as images in your report.

For the normal cyclic prefix used in this exercise, each reference slot contains **14 OFDM symbols**, numbered **0–13**. Use the SIB1 **reference SCS** to construct the common pattern:

| Reference SCS | Reference slot duration |
| --- | ---: |
| 15 kHz | 1 ms |
| 30 kHz | 0.5 ms |
| 60 kHz | 0.25 ms |
| 120 kHz | 0.125 ms |

```text
Number of reference slots in a pattern = Pattern period ÷ Reference slot duration
```

For each pattern:

1. Place `nrofDownlinkSlots` full DL slots at the beginning.
2. If `nrofDownlinkSymbols` is nonzero, mark that many symbols at the beginning of the next slot as DL.
3. Place `nrofUplinkSlots` full UL slots at the end.
4. If `nrofUplinkSymbols` is nonzero, mark that many symbols at the end of the slot immediately before the full UL slots as UL. If there are no full UL slots, use the last slot of the pattern.
5. Mark all remaining symbols as **F (flexible)**. The partial DL and partial UL portions may be in the same slot or in different slots.

Use **D = downlink**, **U = uplink**, and **F = flexible**. Include a legend, the common 0–5 ms time scale, slot indices, and repeating-period boundaries. You may restart the local slot index at zero at each period boundary, but label the boundaries clearly. Expand any mixed slot to show all 14 symbols. Do not automatically label flexible symbols as UL or guard time.

If `pattern2` is present, draw `pattern1` followed by `pattern2`; the combined sequence repeats. Label each pattern boundary and show the full combined duration.

First, count the symbols over **one complete repeating sequence** for each configuration. The two sequences may have different durations, so their raw UL symbol counts alone cannot be used to compare resources per unit time:

```text
Configured DL symbols = 14 × Full DL slots + Partial DL symbols
Configured UL symbols = 14 × Full UL slots + Partial UL symbols
Flexible symbols = Total symbols - Configured DL symbols - Configured UL symbols
Configured UL symbol share (%) = Configured UL symbols ÷ Total symbols × 100
```

For two patterns, sum the counts and durations of both patterns before calculating the share. These are symbol-count shares of the common configuration, not exact airtime measurements or measured throughput shares.

| TDD setting | Repeating duration (ms) | Total reference slots | DL symbols | UL symbols | Flexible symbols | Configured UL symbol share (%) |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| DL-heavy | 5 | 10 | 104 | 32 | 4 | 22.86 |
| Balanced | 2.5 | 5 | 34 | 32 | 4 | 45.71 |

Calculation (30 kHz SCS → 0.5 ms per slot, 14 symbols per slot):

- DL-heavy: 5 ms ÷ 0.5 ms = 10 slots → 140 symbols. DL = 14 × 7 + 6 = 104; UL = 14 × 2 + 4 = 32; Flexible = 140 − 104 − 32 = 4; UL share = 32 ÷ 140 × 100 = 22.86%.
- Balanced: 2.5 ms ÷ 0.5 ms = 5 slots → 70 symbols. DL = 14 × 2 + 6 = 34; UL = 14 × 2 + 4 = 32; Flexible = 70 − 34 − 32 = 4; UL share = 32 ÷ 70 × 100 = 45.71%.

Both configurations have no `pattern2`, so the repeating sequence is just `pattern1`.

### TDD diagrams (same 0–5 ms axis)

Slot indices are global (0–9) over the 5 ms interval. Dashed lines mark repeating-period boundaries. The mixed slot is expanded: symbols 0–5 = D, 6–9 = F, 10–13 = U.

**DL-heavy (period 5 ms, one period fills the 5 ms):**

![task4-tdd-dl-heavy-diagram](https://hackmd.io/_uploads/HypT3256Mx.png)



**Balanced (period 2.5 ms, repeated twice in 5 ms):**

![task4-tdd-balanced-diagram](https://hackmd.io/_uploads/ByDCnh9aGe.png)



Next, count the symbols over the **same 5 ms interval** shown in your diagrams. Multiply each per-sequence count by the number of complete repetitions in 5 ms. For these captures, 5 ms contains a whole number of repetitions for both configurations.

| TDD setting | Repeating duration (ms) | Repetitions in 5 ms | DL symbols in 5 ms | UL symbols in 5 ms | Flexible symbols in 5 ms |
| --- | ---: | ---: | ---: | ---: | ---: |
| DL-heavy | 5 | 1 | 104 | 32 | 4 |
| Balanced | 2.5 | 2 | 68 | 64 | 8 |

Check: both rows add up to 140 symbols (10 slots × 14) in 5 ms: 104 + 32 + 4 = 140 and 68 + 64 + 8 = 140.

**Question:** Can two patterns have the same UL symbol count per period but provide different numbers of UL symbols per unit time? Explain using your two tables.

**Answer:** Yes. Both configurations have the same 32 UL symbols per repeating period (14 × 2 + 4), but the DL-heavy period lasts 5 ms while the balanced period lasts only 2.5 ms. Over the same 5 ms, DL-heavy therefore contains 32 UL symbols (one repetition) and balanced contains 64 UL symbols (two repetitions). The configured UL symbol share rises from 22.86% to 45.71%, so balanced offers twice as many UL symbols per unit time even though the per-period UL count is identical.

### 4.5 Connect SIB1 to the Performance Results

Use the supplied mapping between the reference configuration captures and scenarios. Complete this table using Tasks 1–3 and your SIB1 analysis:

| Scenario | Reference TDD setting | JSON traffic direction | Configured UL symbol share (%) | Throughput (Mbps) | Packet loss (%) |
| --- | --- | --- | ---: | ---: | ---: |
| A | DL-heavy | downlink | 22.86 | 90.65 |0.44 |
| B | DL-heavy | uplink | 22.86 | 109.12 | 0.24 |
| C | Balanced | uplink | 45.71 | 124.42 | 2.13 |

(For A the UL share is the same cell configuration as B; its traffic is downlink, so the DL symbol share, 74.29%, is the relevant resource for A.)

Answer:

1. Which decoded TDD parameters differ between DL-heavy and balanced?
   Only two parameters differ: `dl-UL-TransmissionPeriodicity` (5 ms vs 2.5 ms) and `nrofDownlinkSlots` (7 vs 2). The reference SCS (30 kHz), `nrofDownlinkSymbols` (6), `nrofUplinkSlots` (2), `nrofUplinkSymbols` (4) are the same, and neither has `pattern2`.
2. Which configuration provides more configured UL symbols per unit time? Support your answer with the diagram and calculated share.
   Balanced. Each period has 32 UL symbols in both cases, but balanced repeats every 2.5 ms, so it has 64 UL symbols per 5 ms against 32 for DL-heavy. The UL share is 45.71% (32 ÷ 70) for balanced and 22.86% (32 ÷ 140) for DL-heavy; in the diagrams, UL slots appear twice in 5 ms for balanced and once for DL-heavy.
3. How does that difference help explain the B/C throughput comparison?
   C has twice as many configured UL symbols per unit time as B, so the UE has more uplink transmission opportunities, which is consistent with C's higher uplink throughput (124.42 vs 109.12 Mbps, +14.02%). The gain is much smaller than the doubling of UL symbols, so other factors (scheduling, radio conditions, overhead, offered traffic) also limit throughput, and this comparison does not prove that TDD is the only cause.
4. Why can A and B use the same SIB1 TDD configuration even though their traffic directions differ?
   The SIB1 `tdd-UL-DL-ConfigurationCommon` is a cell-wide setting that defines which symbols are DL, UL or flexible; it does not depend on the application traffic. The traffic direction is chosen by the iperf3 `reverse` setting, so the same DL-heavy cell carries downlink in A and uplink in B, and the scheduler uses the DL or UL resources accordingly.

> More configured UL resources can provide more opportunities for uplink traffic, but they do not guarantee a proportional throughput increase. Scheduling, radio conditions, overhead, offered traffic, and other radio settings also affect the result. This comparison does not establish that the TDD pattern is the only cause of a performance difference. SIB1 does not show actual slot-by-slot data scheduling. Because the configuration captures were recorded separately, use them as references for the supplied settings, not as independent proof of what was broadcast during each original performance run.

### Optional Observation — PRACH Configuration Index

In the same SIB1, find:

```text
servingCellConfigCommon
  → uplinkConfigCommon
    → initialUplinkBWP
      → rach-ConfigCommon
        → setup
          → rach-ConfigGeneric
            → prach-ConfigurationIndex
```

Wireshark may omit or display the `setup` layer differently. You can also use this display filter, then confirm the field is inside SIB1:

```text
nr-rrc.prach_ConfigurationIndex
```

| TDD setting | SIB1 packet number | PRACH configuration index | Proof                                                                       |
| ----------- | ------------------:| -------------------------:| --------------------------------------------------------------------------- |
| DL-heavy    |                  2 |                        98 | ![task4-heavy-optional-index](https://hackmd.io/_uploads/ry6-an5Tzx.png)    |
| Balanced    |                  2 |                        98 | ![task4-balanced-optional-index](https://hackmd.io/_uploads/rJe7639pfx.png) |

PRACH is used to send a random-access preamble. Its configuration index is not the DL/UL slot ratio and does not directly explain iperf3 throughput. Equal indices in the two configurations are a valid observation. Drawing PRACH occasions requires additional parameters and the applicable TS 38.211 table; it is outside the required TDD drawing task.

## 7. Task 5 — Is the TDD Setting Suitable?

Complete the table using the supplied scenario mapping, the SIB1 reference configurations from Task 4, and your throughput results:

| Scenario | TDD setting | Traffic direction | Is the setting suitable for this direction? Why? |
| --- | --- | --- | --- |
| A | DL-heavy | Downlink | Yes. DL-heavy gives 104 of 140 symbols (74.29%) to downlink in every 5 ms, which matches the downlink traffic. Measured throughput was 90.65 Mbps. |
| B | DL-heavy | Uplink | Not ideal. It has only 32 UL symbols per 5 ms (22.86%), so uplink has fewer opportunities. It still reached 109.12 Mbps with low loss (0.24%), but it is the most limited uplink setting of the two. |
| C | Balanced | Uplink | More suitable than DL-heavy. It has 64 UL symbols per 5 ms (45.71%), twice as many as B, and reached 124.42 Mbps, but with higher loss (2.13%) and jitter. |

Use these ideas:

- More DL resources provide more downlink transmission opportunities.
- More UL resources provide more uplink transmission opportunities.
- A DL-heavy setting is suitable for downlink traffic.
- A DL-heavy setting may limit heavy uplink traffic.
- A balanced setting gives uplink traffic more opportunities than the DL-heavy setting in this lab.

> The configuration captures show the reference broadcast TDD settings. The iperf3 JSON reports performance; it does not contain TDD parameters. The scenario captures are used for traffic analysis and need not include SIB1.

**Question:** Why might Scenario C achieve better uplink throughput than Scenario B? Refer to your SIB1 diagrams and measured results.

**Answer:** In the diagrams, the balanced configuration repeats its UL slots every 2.5 ms, giving 64 UL symbols (45.71%) in 5 ms, while the DL-heavy configuration has only 32 UL symbols (22.86%) in the same time. With twice as many uplink opportunities, C can send more data, and its throughput is 124.42 Mbps compared with 109.12 Mbps for B (+14.02%). The improvement is far smaller than the 2× increase in UL symbols and comes with higher packet loss (2.13% vs 0.24%), so other factors such as scheduling, radio conditions and the offered iperf3 rate also matter. Note also that A's downlink throughput (90.65 Mbps) is lower than B's uplink throughput even though DL-heavy favors downlink, so TDD share alone does not determine throughput.

## 8. Task 6 — Find GTP-U Information

Open a **scenario capture** in Wireshark and enter this display filter:

```text
gtp && ip
```

Choose a packet that contains two IPv4 headers. In the packet details, expand:

1. the first `Internet Protocol Version 4` header;
2. `User Datagram Protocol`;
3. `GPRS Tunneling Protocol`; and
4. the second `Internet Protocol Version 4` header.

Record the following information from one packet:

| Item                      | Value                        |
| ------------------------- | ---------------------------- |
| Scenario capture filename | scenario-a-dl-heavy-downlink.pcapng |
| Packet number             | 1892                         |
| Outer source IP           | 192.168.70.134               |
| Outer destination IP      | 192.168.70.129               |
| Inner source IP           | 192.168.70.135               |
| Inner destination IP      | 10.0.0.6                     |
| TEID                      | 0xcddc4529 (3453764905)      |

Proof


| scenario-a-first ipv4 and udp                                                     | scenario-a-second ipv4 and gprs                                                     |
| --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| ![task6-scenario-a-first ipv4 and udp](https://hackmd.io/_uploads/SkIdah9Tfl.png) | ![task6-scenario-a-second ipv4 and gprs](https://hackmd.io/_uploads/S1YAT39pGg.png) |
                              
 

Meaning:

- **Outer IP addresses:** The GTP-U tunnel endpoints.
- **Inner IP addresses:** The original UE/application packet.
- **TEID:** Identifies the GTP-U tunnel at the receiving endpoint.

Take one screenshot showing the selected packet and these fields.

> Some GTP packets are echo or control-related packets. Select one that contains an inner IPv4 packet.

## 9. Task 7 — TCP Retransmissions

Use the **scenario captures** for this task.

The performance traffic is UDP, but iperf3 also uses a small TCP control connection.

Apply this Wireshark filter:

```text
tcp.analysis.retransmission
```

Record the number of displayed packets:

| Scenario | Displayed TCP retransmissions |
| -------- | -----------------------------:|
| A        |                            14 |
| B        |                            12 |
| C        |                            12 |

proof
| Scenario |                                                     screenshot |
| -------- | --------------------------------------------------------------:|
| A        | ![task7-scenario-a](https://hackmd.io/_uploads/BkM-Rhcpfx.png) |
| B        | ![task7-scenario-b](https://hackmd.io/_uploads/H1EGR29pze.png) |
| C        | ![task7-scenario-c](https://hackmd.io/_uploads/ByPm035aGg.png) |


> Report the number displayed by Wireshark using the specified filter. Counts may vary with Wireshark analysis settings. Include screenshots showing the filter and displayed count for each scenario. The filter includes any TCP flows that Wireshark can decode in the capture; a displayed retransmission is not, by itself, proof of loss in the UDP test.

**Question:** Do these TCP retransmissions directly represent lost UDP performance packets?

**Answer:** No. The retransmissions are on the small TCP control connection that iperf3 uses to set up and report the test, while the performance data is sent over UDP, which has no retransmission. UDP loss is calculated by iperf3 from gaps in the UDP sequence numbers (`lost_percent`). The counts also do not follow the loss: B and C both show 12 retransmissions, although their UDP loss is 0.24% and 2.13%.

## 10. Task 8 — RTT and Packet Interval

Use a **scenario capture** for both parts of this task.

### TCP RTT

Apply:

```text
tcp.analysis.ack_rtt
```

Select one packet from the iperf3 TCP control connection and record its ACK RTT:

| Scenario | Packet number | Example TCP ACK RTT |
| -------- | -------------:| ------------------- |
| A        |          1842 | 18.162000 ms        |

proof
| Scenario | screenshot                                                         |
| -------- | ------------------------------------------------------------------ |
| A        | ![task8-scenario-a-rtt](https://hackmd.io/_uploads/Byp40h56Ge.png) |

This value is the RTT of the TCP control connection. It is not the RTT of the UDP data packets.

### Packet Interval

Apply a display filter to select UDP data packets from the same flow.

Select two consecutive, distinct UDP data packets in the filtered packet list. Avoid duplicated observations caused by merged captures; use the same capture point/interface when possible.

In the packet details of the second packet, find:

```text
Time delta from previous displayed frame
```

Record the scenario, display filter, packet numbers, and time interval.

| Scenario | Display filter       | Packet numbers | Time interval |
| -------- | -------------------- | -------------- | ------------- |
| A        | udp.dstport == 51624 | 1892 → 1896    | 1.045000 ms   |

proof
| Scenario | screenshot                                                                     |
| -------- | ------------------------------------------------------------------------------ |
| A        | ![task8-scenario-a-packet interval](https://hackmd.io/_uploads/S1NU02qazl.png) |


**Note:** The time interval depends on the applied display filter. Make sure the selected packets are distinct data packets, not duplicated observations from different capture points. Do not use their interval to infer the TDD slot pattern.

## 11. Final Questions

Answer each question in two or three sentences:

1. Does Scenario A use a TDD setting that matches its traffic direction?
2. Why is Scenario B expected to have limited uplink resources? Cite a parameter or count from the DL-heavy SIB1 reference configuration.
3. How much does uplink throughput change from B to C?
4. What negative result, if any, is visible in C? Support your answer with packet loss or jitter values.
5. Why can increasing UL resources improve uplink throughput?
6. Is adding more UL resources always the best solution? Explain using throughput and packet loss.
7. Suggest one simple follow-up test that could improve throughput without producing too much packet loss.
8. What can the configuration captures tell you that the iperf3 JSON cannot? Why are separately recorded configuration captures not direct proof of the configuration during an earlier test?

### Answers

1. **Does Scenario A use a TDD setting that matches its traffic direction?** Yes. A is downlink traffic and uses the DL-heavy setting, which has 7 full DL slots plus 6 DL symbols per 5 ms, i.e. 104 of 140 symbols (74.29%) for downlink.
2. **Why is Scenario B expected to have limited uplink resources?** The DL-heavy SIB1 has `nrofUplinkSlots` = 2 and `nrofUplinkSymbols` = 4 in a 5 ms (10-slot) period. That is only 32 UL symbols out of 140 (22.86%), compared with 104 DL symbols.
3. **How much does uplink throughput change from B to C?** It increases from 109.12 Mbps to 124.42 Mbps, about +15.30 Mbps or +14.02%.
4. **What negative result is visible in C?** Packet loss rises from 0.24% to 2.13% (about 9 times higher) and jitter rises from 0.116 ms to 0.162 ms (about 39% higher), so the extra throughput comes with worse transmission quality.
5. **Why can increasing UL resources improve uplink throughput?** More UL symbols per unit time give the UE more transmission opportunities that the scheduler can use for uplink data. In the balanced setting UL slots occur every 2.5 ms instead of every 5 ms, so 64 instead of 32 UL symbols are available in 5 ms.
6. **Is adding more UL resources always the best solution?** No. UL symbols doubled but throughput rose only 14.02%, while packet loss rose from 0.24% to 2.13%. Moving resources to UL also reduces DL (DL symbol share falls from 74.29% to 48.57% in 5 ms), which hurts downlink-heavy traffic.
7. **Follow-up test:** Repeat Scenario C (balanced, uplink) at several lower iperf3 target bitrates, for example between B's and C's achieved rates (about 110 to 125 Mbps), and repeat each run a few times. Then choose the highest rate whose packet loss stays low (for example below 1%).
8. **What can the configuration captures show that the JSON cannot?** They show the actual SIB1 TDD parameters (SCS, period, DL/UL slot and symbol counts), while the JSON only reports traffic direction and performance. Because they were recorded separately during initial cell access and the scenario mapping was supplied by the instructor, they show the reference settings, not what the cell was broadcasting during each original iperf3 run.

## 12. Submission Checklist

Submit one Markdown file containing:

- the completed direction table;
- the throughput table and B-to-C calculation;
- the B/C packet-loss and jitter table;
- the SIB1 TDD parameter table, including filenames and packet numbers;
- one SIB1 TDD screenshot per configuration, plus `pattern2` screenshots if needed;
- two TDD diagrams: DL-heavy and balanced, with D/U/F labels and mixed-slot symbol details;
- the per-sequence symbol-count / UL-share table, the common 5 ms comparison table, and the configuration-to-performance table;
- the Task 4.5 answers and the TDD suitability table;
- one GTP-U screenshot and its completed table;
- the TCP retransmission counts and screenshots;
- one TCP ACK RTT example;
- one packet-interval example; and
- answers to the final questions.

The PRACH observation is optional and carries no required points. Include all referenced image files alongside the Markdown file, or submit a ZIP containing the report and images.

## 13. Grading Rubric

| Item | Points |
| --- | ---: |
| Traffic direction | 5 |
| Throughput values and calculation | 20 |
| B/C packet-loss and jitter comparison | 15 |
| SIB1 TDD parameter extraction and screenshot evidence | 10 |
| TDD diagrams, symbol counts, and UL shares | 10 |
| TDD suitability and connection to measured performance | 10 |
| GTP-U fields and screenshot | 15 |
| TCP retransmission, RTT, and packet interval | 10 |
| Clear final answers | 5 |
| **Total** | **100** |

## References

These references support the field names and interpretation. Reading the full specifications is not required.

- [Wireshark NR RRC display filter reference](https://www.wireshark.org/docs/dfref/n/nr-rrc.html): SIB1 and TDD display-filter fields.
- [3GPP TS 38.331, Release 18](https://www.etsi.org/deliver/etsi_ts/138300_138399/138331/18.08.00_60/ts_138331v180800p.pdf): SIB1, TDD-UL-DL-ConfigCommon, and TDD-UL-DL-Pattern definitions.
- [3GPP TS 38.213, Release 17, Section 11.1](https://www.etsi.org/deliver/etsi_ts/138200_138299/138213/17.06.00_60/ts_138213v170600p.pdf): Interpretation of common TDD slot and symbol allocation.
