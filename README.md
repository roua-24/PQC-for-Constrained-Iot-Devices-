# Post-Quantum Cryptography for Constrained IoT Devices: Embedded Benchmarking and Network-Resilience Study

An independent study of post-quantum key-establishment schemes on a simulated ARM Cortex-M4 microcontroller, followed by an experimental extension that measures how post-quantum cryptography behaves in a lightweight IoT protocol (CoAP) when the network loses packets or adds delay.

Author: Rouaa jendoubi
Contact: rouaajendoubi@gmail.com    
Status: in progress since 09/2026. 

---

## 1. Summary for readers in a hurry

| Question | Short answer |
|---|---|
| What is the topic? | Whether post-quantum cryptography (PQC) is practical on small, low-power IoT devices, and how it behaves on unreliable networks. |
| What is done here? | (1) Set up a standard benchmarking framework (pqm4) on a simulated Cortex-M4. (2) Reproduce published measurements for Kyber and Classic McEliece. (3) Extend the work to CoAP under packet loss and latency. |
| What is new compared with the literature? | The extension in step 3. A 2025 systematic review reports that resilience of PQC under degraded network conditions in lightweight IoT protocols is not yet covered. Steps 1 and 2 are reproduction work, not new research. |
| What hardware is used? | None. Everything runs in emulation (Renode or QEMU) and on a Linux virtual machine. The consequences of this choice are stated in section 8. |
| Why does it matter for a telecom or security master's? | It connects network behaviour (loss, delay, retransmission) with cryptographic cost, which is a question at the boundary of networks and security. |

---

## 2. Current status

This section is updated whenever work is completed. Nothing is listed as completed unless the corresponding result is in this repository.

| Part | Content | Status |
|---|---|---|
| 1 | Literature review and gap analysis | [Planned / In progress / Completed] |
| 2 | Benchmarking toolchain (pqm4) on a simulated Cortex-M4 | [Planned / In progress / Completed] |
| 3 | Reproduction of Kyber and Classic McEliece measurements | [Planned / In progress / Completed] |
| 4 | Energy cross-validation against published values | Deferred (needs physical hardware for a meaningful result) |
| 5 | CoAP with PQC under packet loss and latency | [Planned / In progress / Completed] |

Last updated: [date]

---

## 3. Background and motivation

### 3.1 Why post-quantum cryptography

Public-key schemes used today for key exchange (RSA, elliptic-curve Diffie-Hellman) would be broken by a sufficiently large quantum computer running Shor's algorithm. Encrypted traffic recorded now could be decrypted later ("harvest now, decrypt later"), which matters for devices that stay in the field for ten years or more, such as industrial sensors and smart meters. NIST has standardised a lattice-based key-encapsulation mechanism derived from Kyber (ML-KEM) for this purpose.

### 3.2 Why IoT devices are a difficult case

PQC schemes generally need more computation, more memory and larger messages than the classical schemes they replace. Constrained microcontrollers have little RAM and flash, run at low clock speeds and are often battery-powered. Their networks (low-power wireless, cellular IoT) lose packets and add delay. Whether PQC fits within these limits is therefore an open engineering question and not only a cryptographic one.

### 3.3 The two algorithms studied

| Algorithm | Family | Why it is included |
|---|---|---|
| Kyber (standardised as ML-KEM) | Lattice-based key encapsulation | The main NIST choice. Relatively small keys and fast operations. |
| Classic McEliece | Code-based key encapsulation | Very conservative security record, but public keys are very large. It is a useful extreme case for memory and message size. |

Both are used for key establishment (key generation, encapsulation, decapsulation). Signature schemes are outside the core scope.

---

## 4. Research gap and objectives

### 4.1 Gap

The starting point is a 2025 systematic review of PQC resilience in lightweight IoT protocols (reference [1]). Two related studies integrate PQC into CoAP and MQTT-SN and measure handshake time, message size and end-to-end time on small devices (references [6] and [7]). According to the review and to the studies read for this project, none of them tests behaviour under degraded network conditions such as packet loss and added latency. This statement reflects the literature available when the review was written. Before the final report, newer publications are checked again, and the gap is restated or removed if it has been closed.

### 4.2 Objectives

1. Build a documented, reproducible benchmarking environment for PQC on a simulated Cortex-M4 target.
2. Reproduce published cost measurements (CPU cycles, memory) for Kyber and Classic McEliece and explain every difference between my numbers and the published ones.
3. Measure how a PQC-enabled CoAP exchange behaves under controlled packet loss and latency, and compare it with a classical-cryptography baseline under the same conditions.

### 4.3 Research question for the extension

Does replacing classical key establishment with a post-quantum scheme make a CoAP session more fragile on a lossy or delayed network, and if so by how much?

---

## 5. Method, step by step

Each part below states its purpose, the inputs, what is done, the output, and how the result is checked. This is written so that a reader can follow the reasoning without running anything.

### Part 1: Literature review and gap analysis

- Purpose: make sure the later experiments address a real gap and do not repeat existing work.
- Input: the systematic review [1], the two protocol-integration studies [6] and [7], and the measurement papers [3] and [4].
- Work: extract the comparison table from the review, identify the studies closest to this project (Kyber, embedded targets, CoAP and MQTT-SN), and write one paragraph per gap stating why it exists and what is still missing.
- Output: `docs/gap_analysis.md`.
- Check: each claim in the gap analysis is cross-checked against at least one independent paper, and a search for newer publications is done before the analysis is finalised.

### Part 2: Benchmarking environment (pqm4 on a simulated Cortex-M4)

- Purpose: obtain a standard, comparable way to measure PQC cost on a microcontroller. pqm4 [2] is a widely used framework that benchmarks NIST candidates on an ARM Cortex-M4. Its original target is the STM32L4 family.
- Input: the pqm4 repository, an ARM cross-compiler (`arm-none-eabi-gcc`), and an emulator (Renode, with QEMU as an alternative).
- Work: install the toolchain, build one scheme (Kyber512), run its benchmark in the emulator, and compare the cycle count with the published order of magnitude. Then add a second scheme to confirm that the setup is not specific to one case.
- Output: `docs/setup_notes.md` with every command and version needed to repeat the setup, plus a first benchmark table.
- Check: the cycle counts for Kyber512 must be in the same order of magnitude as those published for the same target. A difference of ten times or more is treated as a configuration error (clock frequency, compiler flags) and fixed before continuing.

### Part 3: Reproduction of a published comparison (Kyber and Classic McEliece)

- Purpose: test whether a published result can be reproduced independently, and learn where and why it cannot. The reference is Alnaseri et al. [3], which compares the two schemes on embedded systems.
- Input: the tooling from Part 2, the Kyber implementation in pqm4, and a Classic McEliece implementation. If pqm4 does not include Classic McEliece for this target, an alternative implementation (for example from PQClean or liboqs) is used and this is documented.
- Work: identify the exact parameter sets used in the paper (NIST security level), run key generation, encapsulation and decapsulation for each scheme, with at least 10 to 20 runs per case so that variance can be reported. Record CPU cycles and memory.
- Energy: real energy cannot be measured in emulation. An estimate is computed as cycles multiplied by supply voltage multiplied by average current from the datasheet of the simulated microcontroller. It is labelled "estimated" everywhere it appears and is never presented as a measurement.
- Output: `results/reproduction/` containing the raw data, the benchmark scripts and a table laid out like the original paper's, with a section that lists what matches, what differs and a hypothesis for each difference.
- Check: published and reproduced values are compared figure by figure. The very large Classic McEliece public key is a known point and serves as a sanity check on the parameter set used.

### Part 4: Energy cross-validation (deferred)

- Purpose: compare the energy estimate from Part 3 with two independent published energy studies ([4] and [5]).
- Why deferred: both studies measure energy on real boards, which cannot be reproduced in emulation. Without a physical board the comparison would be a datasheet estimate against real measurements and would add little. This part is carried out only if hardware becomes available. The limitation is stated openly in section 8.

### Part 5: Extension, PQC in CoAP under network degradation

- Purpose: the original contribution. Measure whether PQC makes CoAP more sensitive to packet loss and delay.
- Background: CoAP is a lightweight request-response protocol for constrained devices and is secured with DTLS. PQC key exchange produces larger handshake messages, which may be split into several packets, so each lost packet can cost more.
- Input: wolfSSL built with post-quantum support, the libcoap library, and Linux traffic control (`tc netem`) to inject controlled loss and delay.
- Work:
  1. Build wolfSSL with PQC support and check that a basic DTLS handshake with Kyber succeeds before touching the application protocol.
  2. Integrate it with libcoap and check a plain CoAP exchange end to end.
  3. Reproduce the published behaviour on an undisturbed network first. Otherwise it is impossible to tell whether a later problem comes from the PQC set-up or from the network degradation.
  4. Run the test matrix in section 6.
- Output: `results/coap_resilience/` with raw logs, the analysis scripts and the final tables and plots.
- Check: results under no degradation are compared with published figures. Every configuration is run several times and variance is reported.
- Fallback if time is short: compare two PQC algorithms under the same degraded conditions instead of adding a second protocol (MQTT-SN).

---

## 6. Experiment design for Part 5

| Factor | Levels |
|---|---|
| Key establishment | Classical (elliptic-curve baseline) and post-quantum (Kyber) |
| Packet loss | 0 %, 5 %, 15 % |
| Added one-way delay | [values to be fixed, for example none and one or two non-zero levels] |
| Repetitions per configuration | [number, at least 20] |

Metrics recorded for every run:

- Whether the handshake completed (success rate).
- Time to complete the handshake.
- Number of retransmissions.
- Total bytes sent during the handshake, and whether messages were fragmented.
- Time for a complete request and response after the handshake.

Example of how degradation is injected (the interface name depends on the set-up):

```
sudo tc qdisc add dev <interface> root netem delay 50ms loss 5%
sudo tc qdisc del dev <interface> root
```

---

## 7. Results

Results are added only after they have been produced by the scripts in this repository. Empty cells mean "not yet measured".

### 7.1 Reproduction of the embedded benchmarks (Part 3)

| Scheme | Operation | Cycles (mean, n = [ ]) | Std. dev. | RAM | Flash | Published value | Comment |
|---|---|---|---|---|---|---|---|
| Kyber512 | Key generation | | | | | | |
| Kyber512 | Encapsulation | | | | | | |
| Kyber512 | Decapsulation | | | | | | |
| Classic McEliece [parameter set] | Key generation | | | | | | |
| Classic McEliece [parameter set] | Encapsulation | | | | | | |
| Classic McEliece [parameter set] | Decapsulation | | | | | | |

### 7.2 Estimated energy (Part 3)

All values are estimates computed from cycles and datasheet current. They are not measurements.

| Scheme | Operation | Estimated energy | Assumptions |
|---|---|---|---|
| | | | |

### 7.3 CoAP under degraded conditions (Part 5)

| Key establishment | Loss | Delay | Handshake success rate | Median handshake time | Retransmissions | Bytes sent |
|---|---|---|---|---|---|---|
| Classical | 0 % | | | | | |
| Post-quantum | 0 % | | | | | |
| Classical | 5 % | | | | | |
| Post-quantum | 5 % | | | | | |
| Classical | 15 % | | | | | |
| Post-quantum | 15 % | | | | | |

### 7.4 Discussion

To be written after the results are available. It will state what the data supports, what it does not, and which differences from the literature remain unexplained.

---

## 8. Limitations

These are stated in advance so that results are read correctly.

- **No physical board.** Cycle counts come from an emulator and may differ from real hardware because of timing, memory-system and peripheral behaviour. Only the order of magnitude is expected to agree with published figures.
- **No real energy measurement.** Energy is an estimate from datasheet values. It cannot replace a measurement with a current sensor or oscilloscope.
- **Simulated network.** `tc netem` models loss and delay in a controlled way but does not reproduce interference, congestion or the behaviour of a real low-power wireless link.
- **One protocol and a limited set of algorithms.** The extension covers CoAP, with MQTT-SN and further algorithms only if time allows. Conclusions are not claimed for other protocols.
- **Independent student project.** The work follows published methods and has not been peer reviewed.

Open risks: Classic McEliece may not be available in pqm4 for this target (an alternative implementation is then used), and integrating PQC-enabled DTLS into libcoap may take longer than planned.

---

## 9. Repository structure

```
.
├── README.md
├── docs/
│   ├── gap_analysis.md        Part 1: literature review and gaps
│   └── setup_notes.md         Part 2: how to reproduce the environment
├── src/                       Benchmark and experiment scripts
├── results/
│   ├── reproduction/          Part 3: raw data and tables
│   └── coap_resilience/       Part 5: raw logs, scripts and plots
└── LICENSE
```

---

## 10. Tools and versions

| Tool | Purpose | Version used |
|---|---|---|
| pqm4 | Benchmarking framework for Cortex-M4 | [commit or version] |
| arm-none-eabi-gcc | Cross-compiler | [version] |
| Renode or QEMU | Emulation of the target | [version] |
| wolfSSL (with PQC support) | DTLS with post-quantum key exchange | [version] |
| libcoap | CoAP implementation | [version] |
| tc netem | Packet loss and delay injection | [kernel version] |
| Python | Analysis scripts | [version] |

---

## 11. References

[1] "Resilience of Post-Quantum Cryptography in Lightweight IoT Protocols: A Systematic Review", MDPI Engineering, 2025. https://doi.org/10.3390/eng6120346

[2] M. J. Kannwischer, J. Rijneveld, P. Schwabe, K. Stoffelen, "pqm4: Testing and Benchmarking NIST PQC on ARM Cortex-M4", 2019. https://eprint.iacr.org/2019/844

[3] Alnaseri, Himeur, Atalla, Mansoor, "Complexity of post-quantum cryptography in embedded systems and its optimization strategies", IWCMC 2025, pp. 776-781.

[4] Halak, Gibson, Henley, Botea, Heath, Khan, "Evaluation of performance, energy, and computation costs of quantum-attack resilient encryption algorithms for embedded devices", IEEE Access, vol. 12, pp. 8791-8805, 2024.

[5] "Energy Consumption Framework and Analysis of Post-Quantum Key-Generation on Embedded Devices", arXiv:2505.16614.

[6] Blanco-Romero, Lorenzo, Almenares, Sanchez, Campo, Garcia Rubio, "Integrating Post-Quantum Cryptography into CoAP and MQTT-SN Protocols", IEEE ISCC 2024.

[7] "MQTT Across a Raspberry Pi 5 IoT Network Utilizing Quantum-resistant Signature Algorithms", arXiv:2605.13698.

[8] "pqm4: Benchmarking NIST Additional Post-Quantum Signature Schemes on Microcontrollers", 2024. https://eprint.iacr.org/2024/112

Reference details are as I recorded them while reading. Check titles, authors and identifiers against the original sources before citing them elsewhere.

---

## 12. Reproducing this work

1. Follow `docs/setup_notes.md` to install the toolchain and emulator.
2. Run the benchmark scripts in `src/` and compare the output with the tables in section 7.
3. For Part 5, follow the instructions in `results/coap_resilience/` to build wolfSSL and libcoap, then run the test matrix from section 6.

Exact commands are added to the setup notes as each part is completed, so that nothing in them is untested.

---

## 13. Licence and use

[Licence, for example MIT.] This repository is educational and exploratory. The code is not intended for production use and has not been security-audited.
