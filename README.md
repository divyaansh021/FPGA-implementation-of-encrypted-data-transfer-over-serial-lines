# Secure High-Speed RF Communication on RFSoC 4x2

> PAM Modulation · Lattice Cryptography · Hardware Homomorphic Addition

---

## Overview

This project implements a **secure RF communication system** on the Xilinx RFSoC 4x2 development board. It combines multiple PAM modulation schemes with lattice-based post-quantum cryptography and a custom hardware homomorphic adder IP — all running in real-time on FPGA.

The key achievement is **BER = 0** across all modulations and modes, including encrypted transmission and hardware homomorphic addition, at both **614 MHz** and **1 GHz** carrier frequencies.

---

## System Pipeline

```
Python (PS)
    │
    ├── LWE Encrypt ──────────────────────────────────┐
    │                                                  │
    │                                         Hom. Adder IP (FPGA)
    │                                         c_add = (c1+c2) mod q
    │                                                  │
    └──────────────────────────────────────────────────┘
    │
    ▼
Amplitude Controller IP (FPGA)
    │  gain register → 256-bit AXI Stream
    ▼
RFDC DAC → RF at 614.4 MHz / 1 GHz
    │
    ▼  (SMA loopback J17 → J15)
    │
RFDC ADC
    │
    ▼
Packet Receiver IP (FPGA)
    │  packet_done flag
    ▼
Python (PS)
    │
    └── LWE Decrypt → original message ✅
```

---

## Hardware

| Parameter | Value |
|-----------|-------|
| Board | RFSoC 4x2 |
| Device | xczu48dr-ffvg1517-2-e |
| DAC Sample Rate | 6553.6 MSPS |
| ADC Sample Rate | 4915.2 MSPS |
| Framework | PYNQ |
| Loopback | J17 → J15 (SMA) |

---

## PAM Modulations

| Mode | Levels | Bits/Symbol |
|------|--------|-------------|
| OOK | 2 | 1 |
| PAM4 | 4 | 2 |
| PAM8 | 8 | 3 |
| PAM16 | 16 | 4 |

---

## Custom Hardware IPs

### 1. Amplitude Controller
- Scales signal amplitude for each PAM level
- AXI Lite register map: gain (0–255), max_val, enable
- Output: 256-bit AXI Stream (8 × 32-bit samples)
- Two instances: ch0 @ `0x500000000`, ch1 @ `0x4800000000`

### 2. Packet Receiver
- Detects end of received packet via TLAST
- Sets sticky `packet_done` flag readable by Python via GPIO
- MMIO: `0x80060000`

### 3. Homomorphic Adder
- Computes `c_add[i] = (c1[i] + c2[i]) mod q` for i = 0..4
- 5 integers processed in **parallel** in hardware
- PL computation: ~10 ns
- Total latency (including AXI): ~184 µs
- MMIO: `0x800A0000`

---

## Cryptography

**Scheme:** Learning With Errors (LWE) — post-quantum lattice cryptography

| Parameter | Value |
|-----------|-------|
| Modulus q | 127 (prime) |
| Vector length L | 5 |
| Representation | Centered [−63, 63] |

**Encrypt:** `c = encode(message) + ε`  
**Decrypt:** `m = decode(c mod q)`  
**HomAdd:**  `decrypt(c1 + c2) = decrypt(c1) + decrypt(c2)`

---

## Results

### BER Results — 614 MHz

| Mode | Packets Tested | BER |
|------|---------------|-----|
| OOK | 100 | 0.0000 ✅ |
| PAM4 | 100 | 0.0000 ✅ |
| PAM8 | 100 | 0.0000 ✅ |
| PAM16 | 100 | 0.0000 ✅ |
| Encrypted RF | 200 | 0.0000 ✅ |
| HomAdd (all modes) | 40 each | 0.0000 ✅ |

### BER Results — 1 GHz

| Mode | Packets Tested | BER |
|------|---------------|-----|
| OOK | 40 | 0.0000 ✅ |
| PAM4 | 40 | 0.0000 ✅ |
| PAM8 | 40 | 0.0000 ✅ |
| PAM16 | 40 | 0.0000 ✅ |
| HomAdd (all modes) | 40 each | 0.0000 ✅ |

### Homomorphic Adder Timing

| Implementation | Time | Speedup |
|----------------|------|---------|
| Pure software (Python) | 1.230 ms | 1.0× |
| **Hardware IP (optimised)** | **184 µs** | **6.7×** |

---

## Repository Contents

```
├── notebooks/
│   ├── pam.ipynb         # OOK, PAM4, PAM8, PAM16 test
│   ├── encrypted_final.ipynb           # LWE encryption + RF transmission



└── README.md
```

> **Note:** Vivado project files, bitstreams, and custom IP Verilog source are not included in this repository. If you need access to the hardware implementation details, please contact me directly (see below).

---

## Requirements

### Hardware
- Xilinx RFSoC 4x2 Development Board
- SMA cable for loopback (J17 → J15)

### Software
- PYNQ framework v2.7+
- Python 3.8+
- Jupyter Notebook

### Python Packages
```bash
pip install numpy matplotlib pynq
```

---

## Running the Notebooks

1. Copy notebooks to the RFSoC 4x2 board:
```bash
scp notebooks/*.ipynb xilinx@<board-ip>:/home/xilinx/jupyter_notebooks/
```

2. Open Jupyter on the board:
```
http://<board-ip>:9090
```

3. Run notebooks in this order:
   - `pam.ipynb` — verify basic modulation
   - `encrypted_final.ipynb` — test encrypted transmission
  

> **Note:** The bitstream (`base.bit` and `base.hwh`) must be present on the board before running any notebook.

---

## Key Findings

- **BER = 0** achieved across all modulations at 614 MHz and 1 GHz
- Hardware homomorphic addition is **6.7× faster** than software
- LWE lattice cryptography works correctly end-to-end with RF transmission

---

## Contact

This repository contains only the Python notebooks and result plots. The full hardware implementation includes:

- Custom Verilog IP cores (Amplitude Controller, Packet Receiver, Homomorphic Adder)
- Vivado block design and constraints
- Bitstream files
- Timing analysis reports

If you are interested in the complete hardware implementation, want to collaborate, or have questions about the project, feel free to reach out:

📧 **Email:** ee23bt021@iitdh.ac.in  
🔗 **LinkedIn:** linkedin.com/in/divyaansh-dhingra-826117168  
  

---

---

## License

This project is shared for educational and research purposes.  
© 2026 Divyaansh Dhingra. All rights reserved.
