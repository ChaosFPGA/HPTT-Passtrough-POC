# HPTT FPGA Passthrough – Nexys Video Setup

## 🔍 Overview

This project replicates **HPTT-style PCIe passthrough with sideband injection**, using:

- 🎛️ **Digilent Nexys Video** (Artix‑7 XC7A200T, DDR3, FMC)
- 📶 **FMC-to-PCIe adapter** (with redriver/retimer for Gen2/Gen3 integrity)
- 🧵 **PCIe riser cable** (Gen3/4 spec, short length)
- ⚡ Real PCIe devices (e.g. AX210, NVMe SSD) remain 100% functional during stealth TLP injection or trace logging.

Supports passive passthrough, CRC-correct injection, DMA engine (PCILeech-compatible), and potential flow-control spoofing.

---

## 📦 Hardware Requirements

### 🎛️ FPGA Board

- **[Digilent Nexys Video](https://digilent.com/shop/nexys-video-amd-artix-7-fpga-trainer-board-for-multimedia-applications/)**  
  - FPGA: XC7A200T-1SBG484C (Artix‑7)
  - FMC LPC connector (for PCIe passthrough)
  - 1 GB DDR3 (usable as DMA buffer)
  - USB-JTAG + UART interface

### 🧩 FMC PCIe Adapter (Pick One)

| Name                                | Gen | Redriver | Link |
|-------------------------------------|-----|----------|------|
| Terasic PCIe x4 FMC Adapter         | 3   | ✅ Yes   | [Link](https://eu.mouser.com/ProductDetail/Terasic-Technologies/P0492?qs=rrS6PyfT74cJ%252BiEsLp7emg%3D%3D&srsltid=AfmBOoq3SmzcyTZ2VN0o6w_JrGwwTPyU9s_LFsrkHAdJQzbLdFJI1TWD]) |
| Custom Chinese FMC x4 Gen2 Adapter  | 2   | ❌ No    | (AliExpress) Not recommended for stable links |

> ⚠️ **Redriver required** for proper PCIe link training and signal integrity over cables.

### 🧵 Riser Cable

- **PCIe Gen4 x16 Riser Cable**  
  e.g. [Cooler Master Riser Gen4](https://www.coolermaster.com/catalog/coolers/riser-cables/riser-cable-pcie-4-0-x16/) or equivalent.

> Keep under 30 cm. Long risers + no redriver = unstable TLPs or link failures.

### 🔋 Power

- 12 V DC for FPGA + downstream device (e.g. NVMe)
- Optional 3.3 V or 5 V DC for riser or redriver logic

---

## 🧰 Software

| Tool         | Purpose                              | Link |
|--------------|--------------------------------------|------|
| Xilinx Vivado (2022.1+) | Bitstream build & synthesis        | [Xilinx](https://www.xilinx.com/support/download.html) |
| Cable Drivers | USB-JTAG for Nexys Video             | Included in Vivado |
| Serial terminal (e.g. `minicom`, `TeraTerm`) | Runtime UART control | |
| PCILeech (optional) | DMA speed / test framework     | [PCILeech GitHub](https://github.com/ufrisk/pcileech) |

---

## 🧠 Architecture

```
+-------------------------+
|     Host PC (x86_64)    |
+-------------------------+
           |
     PCIe x4/x16 Slot
           |
   [Gen4 PCIe Riser Cable]
           |
+-------------------------+
|   FMC PCIe Adapter      |
|   (with Redriver Chip)  |
+-------------------------+
           |
+-------------------------------+
|    Digilent Nexys Video FPGA  |
| - PCIe PIPE Core              |
| - DMA Engine (PCILeech)       |
| - Injection Controller        |
| - DDR3 Buffer (1 GB)          |
+-------------------------------+ 

```
Injection Core Features:
- CRC-correct TLP injection
- DMA read/write to host memory
- Optional trigger filtering or MMIO snooping
- DDR3 buffering for large payloads

---

## 🔌 Hardware Setup

1. Plug FMC PCIe adapter into **Nexys Video FMC LPC connector**.
2. Connect short PCIe **Gen4 riser cable** to adapter.
3. Plug PCIe device (e.g. **AX210**, **NVMe SSD**) into the adapter’s downstream port.
4. Plug riser into motherboard **x4/x8/x16 slot**.
5. Power the Nexys Video and PCIe device (via external 12 V supply if needed).

---

## 🚀 Theoretical Speed Classification

| Category        | Description                                   | Max Speed            | Supported Hardware               |
| --------------- | --------------------------------------------- | -------------------- | -------------------------------- |
| ⚪ **Basic**     | PCIe Gen1 x1 — limited performance            | ~200 MB/s           | ScreamerM2, CaptainDMA 35T       |
| 🟡 **Standard** | PCIe Gen2 x1 — average stealth DMA capability | ~400 MB/s           | CaptainDMA 75T, Screamer         |
| 🟠 **High**     | PCIe Gen2 x4 — optimal PCILeech performance   | **~1000–1200 MB/s** | ✅ **Nexys Video + Terasic PCA3** |
| 🔴 **Extreme**  | PCIe Gen3 x4 — future-proof high-speed DMA    | ~2000+ MB/s¹        | ZDMAv2 (Kintex/Ultrascale)       |

> ¹ Real-world Gen3 speeds currently limited by PCILeech firmware support.

---

## ✅ Why You’ll Reach ~1000 MB/s with Your Setup

Your build matches or exceeds the ZDMA hardware specs:

- **ZDMA** uses a Xilinx XC7A100T with PCIe Gen2 x4.
- Your **Nexys Video** has a more powerful XC7A200T FPGA.
- Using the **Terasic PCA3 (P0492)** FMC adapter with Gen3 redriver supports clean PCIe Gen2 x4 signaling.
- With proper firmware, your design supports:
  - High-speed DMA reads and writes.
  - PCIe burst TLP support (essential for speeds above 500 MB/s).
  - Compatibility with `pcileech-fpga` (version 4.17+ recommended).

This positions your HPTT build solidly in the **High** performance category, capable of sustained ~1000 MB/s DMA throughput.


### 💻 Programming

1. Open Vivado and load your project or prebuilt `.bit` file.
2. Use **USB-JTAG** to program the Nexys Video FPGA.
3. Monitor UART output via serial terminal at **115200 bps** for logs and injection status.

---

## 🧪 Validation & Testing

Make sure the host system:

- ✅ Enumerates the PCIe device normally
- ✅ Boots and uses the device as expected
- ✅ Is unaffected by passive monitoring or background DMA

Use tools like:

- [`PCILeech`](https://github.com/ufrisk/pcileech) to test DMA capabilities
- `DRVScan` to confirm stealth mode and device enumeration

---


## 🧱 Optional: Enclosure / Mounting

Design a 3D-printable bracket or shell to house:

- Nexys Video board
- FMC PCIe Adapter
- PCIe riser
- PCIe device (e.g. NVMe, Wi-Fi)

> 💡 Add low-speed fans for cooling, especially with high-power devices like NVMe or AX210 under sustained load.

---

## ✅ Confirmed Working Devices

| Device              | Notes                                |
|---------------------|--------------------------------------|
| Intel AX210         | Full passthrough + DMA injection     |
| Samsung 980 Pro NVMe| Stable Gen3 x4 + spoof/trace support |
| Elgato CamLink      | Enumerates and streams correctly     |

---

## 📌 Technical Notes

- TLP injection and CRC patching are **in-band and real-time**
- Link may **downtrain to Gen2** with low-quality risers or long cables
- Use **short, shielded Gen4 risers** and **redriver-based adapters** for best results

---

## 🔐 License

> ⚠️ **FOR EDUCATIONAL PURPOSES ONLY**

Unauthorized use for malicious purposes is strictly prohibited.  
To contribute, request builds, or join development:  
📬 Contact `@ChaosFPGA` on Discord.

---

## 🤝 Credits

- Inspired by the original **HPTT / Deepcover Labs**
- Built using open toolchains: **Xilinx Vivado**, **PCIe PIPE**, and **PCILeech**
- Thanks to the DIY hardware research community for their documentation and open sharing
