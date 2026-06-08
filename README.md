# ZYBO Z7-20 Ethernet — lwIP TCP Echo Server

A bare-metal lwIP TCP echo server running on the ZYBO Z7-20 FPGA board using Vitis 2024.2 with the SDT (Software Design Tools) build flow.

This repository documents the complete bring-up process, including several non-obvious fixes required to get Ethernet working on this board with the new SDT flow.

---

## Hardware

| Item | Detail |
|------|--------|
| Board | Digilent ZYBO Z7-20 |
| SoC | Xilinx Zynq-7000 (XC7Z020) |
| CPU | ARM Cortex-A9 @ 667 MHz |
| PHY | Realtek RTL8211E-VL |
| EMAC | Zynq PS GEM0 (0xE000B000) |
| PHY Interface | RGMII-ID |
| UART | PS UART1 — 115200 baud (second USB COM port) |

## Software

| Item | Detail |
|------|--------|
| IDE | Vitis 2024.2 |
| Build Flow | SDT (Software Design Tools) — `-DSDT` |
| TCP/IP Stack | lwIP 2.2.0 RAW API |
| IP Assignment | DHCP (12-second timeout) → fallback static `192.168.1.10` |
| TCP Port | 7 |

---

## What It Does

The board acts as a **TCP echo server** on port 7. Any data sent to `192.168.1.10:7` is echoed back to the client.

```
PC (192.168.1.x)  ──── TCP :7 ────▶  ZYBO (192.168.1.10)
                  ◀─── echo ──────
```

The application entry point is [`Vitis/lwip_echo_server/src/echo.c`](Vitis/lwip_echo_server/src/echo.c) — specifically `recv_callback()`, which is where you replace the echo logic with your own Ethernet callback.

---

## Repository Structure

```
.
├── Vivado/
│   ├── ZYBO_Z7_20_Ethernet.xpr          # Vivado project file
│   ├── Ethernet_Echo_wrapper.xsa        # Hardware handoff to Vitis
│   └── *.srcs/sources_1/bd/             # Block design (Zynq PS config)
│
├── Vitis/
│   ├── lwip_echo_server/src/            # Application source (main code)
│   │   ├── main.c                       # Network init, DHCP loop, TCP loop
│   │   ├── echo.c                       # TCP echo logic — edit recv_callback()
│   │   └── platform.c                   # Platform init (SCU timer fix inside)
│   │
│   └── platform_ethernet/hw/            # Hardware artifacts
│       ├── Ethernet_Echo_wrapper.xsa    # XSA for BSP generation
│       └── sdt/Ethernet_Echo_wrapper.bit # FPGA bitstream
│
├── patches/
│   ├── xemacpsif_physpeed.c             # Modified PHY speed detection (see Fixes)
│   └── bsp.yaml                         # BSP configuration reference
│
└── troubleshooting_report.html          # Full bring-up report (open in browser)
```

> **Note:** BSP driver sources, FSBL, and all Vitis-generated files are excluded from git. They are regenerated automatically when you build the platform in Vitis.

---

## How to Build

### Prerequisites

- Vivado 2024.2 with Zynq-7000 support
- Vitis 2024.2

### Step 1 — Hardware (Vivado)

The XSA file is already included in the repository (`Vitis/platform_ethernet/hw/Ethernet_Echo_wrapper.xsa`). You can skip this step unless you modify the hardware design.

If you need to regenerate the XSA:
1. Open `Vivado/ZYBO_Z7_20_Ethernet.xpr` in Vivado 2024.2
2. Run **Generate Bitstream**
3. Export hardware: **File → Export → Export Hardware** (include bitstream) → overwrite the XSA in `Vitis/platform_ethernet/hw/`

### Step 2 — Platform (Vitis)

1. Open Vitis 2024.2 and set workspace to `Vitis/`
2. Open the existing `platform_ethernet` platform component
3. Build the platform — this generates the BSP under `ps7_cortexa9_0/`

### Step 3 — Apply Patches

After building the platform, copy the modified PHY driver over the generated one:

```bash
cp patches/xemacpsif_physpeed.c \
   Vitis/platform_ethernet/ps7_cortexa9_0/standalone_ps7_cortexa9_0/bsp/\
libsrc/lwip220/src/lwip-2.2.0/contrib/ports/xilinx/netif/xemacpsif_physpeed.c
```

Then **rebuild the platform** so the patched file is compiled into `libxil.a`.

### Step 4 — Application

1. Open the `lwip_echo_server` application component in Vitis
2. Build (`Ctrl+B`)
3. Program the board: **Run → Run Configurations → Single Application Debug**

---

## Testing

1. Set your PC's Ethernet adapter to a static IP in the same subnet:
   - IP: `192.168.1.100`
   - Subnet: `255.255.255.0`

2. Open a serial monitor on the **second USB COM port** at **115200 baud**. Expected output:

```
-----lwIP TCP echo server ------
Start PHY autonegotiation
Waiting for PHY to complete autonegotiation.
autonegotiation complete
RTL8211E lp_1000(reg10)=0x3800 lp_base(reg5)=0xCDE1
link speed for phy address 1: 1000
DHCP Timeout
Configuring default IP of 192.168.1.10
Board IP: 192.168.1.10
Netmask : 255.255.255.0
Gateway : 192.168.1.1
TCP echo server started @ port 7
```

3. Connect to `192.168.1.10` port `7` via TCP using any client:

   **Packet Sender** (recommended GUI tool):
   - Address: `192.168.1.10` / Port: `7` / Protocol: **TCP**
   - Send any ASCII text → same text echoes back

   **netcat** (Linux/macOS):
   ```bash
   nc 192.168.1.10 7
   ```

   **PowerShell** (Windows):
   ```powershell
   $tcp = New-Object System.Net.Sockets.TcpClient('192.168.1.10', 7)
   $stream = $tcp.GetStream()
   $bytes = [Text.Encoding]::ASCII.GetBytes("hello`n")
   $stream.Write($bytes, 0, $bytes.Length)
   Start-Sleep -Milliseconds 200
   $buf = New-Object byte[] 1024
   $n = $stream.Read($buf, 0, 1024)
   [Text.Encoding]::ASCII.GetString($buf, 0, $n)
   $tcp.Close()
   ```

---

## Fixes Applied

This project required three non-obvious fixes to work on the ZYBO Z7-20 with Vitis 2024.2 SDT. They are documented here so others do not have to rediscover them.

---

### Fix 1 — RTL8211E PHY Speed Detection (patches/xemacpsif_physpeed.c)

**Symptom:** `Phy setup error / Phy setup failure init_emacps`

**Root cause 1 — Stale autoneg latch:**
IEEE 802.3 register 1 bit 5 (Autoneg Complete) is a **latch-high** bit. The FSBL performs autonegotiation during boot and leaves the bit set. The application then reads it, sees "complete", and exits the wait loop immediately — before real autoneg has finished.

**Fix:** Read register 1 once to clear the latch, force `status = 0`, then poll properly:
```c
XEmacPs_PhyRead(xemacpsp, phy_addr, IEEE_STATUS_REG_OFFSET, &status); // clear latch
status = 0;
sleep(1);
while (!(status & IEEE_STAT_AUTONEGOTIATE_COMPLETE)) {
    sleep(1);
    XEmacPs_PhyRead(xemacpsp, phy_addr, IEEE_STATUS_REG_OFFSET, &status);
}
```

**Root cause 2 — RTL8211E reg17 unreliable:**
The RTL8211E vendor-specific PHY Specific Status Register (reg17 / 0x11) returns `0x0000` consistently on this board, making speed detection impossible.

**Fix:** Use standard IEEE autoneg result registers instead:

| Register | Offset | Bits | Meaning |
|----------|--------|------|---------|
| 1000BASE-T LP Status | reg10 (0x0A) | `0x0C00` | Link partner supports 1 Gbps |
| LP Base Ability | reg5 (0x05) | `0x0180` | Link partner supports 100 Mbps |
| LP Base Ability | reg5 (0x05) | `0x0060` | Link partner supports 10 Mbps |

```c
u16_t lp_1000, lp_100_10;
XEmacPs_PhyRead(xemacpsp, phy_addr, IEEE_PARTNER_ABILITIES_3_REG_OFFSET, &lp_1000);
XEmacPs_PhyRead(xemacpsp, phy_addr, IEEE_PARTNER_ABILITIES_1_REG_OFFSET, &lp_100_10);

if      (lp_1000   & 0x0C00) return 1000;
else if (lp_100_10 & 0x0180) return 100;
else if (lp_100_10 & 0x0060) return 10;
```

---

### Fix 2 — DHCP Timeout Loop Never Exits (Vitis/lwip_echo_server/src/platform.c)

**Symptom:** System hangs indefinitely after `link speed for phy address 1: 1000`. Waiting 12+ seconds produces no output.

**Root cause:** The DHCP wait loop in `main.c` relies on `dhcp_timoutcntr` being decremented by a 50 ms timer interrupt. In SDT mode, `platform_enable_interrupts()` is excluded (`#ifndef SDT`), so the timer must be set up in `platform.c`'s `init_timer()`.

`init_timer()` calls `XTimer_SetInterval(50)` and `XTimer_SetHandler(...)` from the xiltimer library. However, the BSP is generated with `XILTIMER_tick_timer = None`, which results in `xtimer_config.h` defining `XTIMER_NO_TICK_TIMER = 1`. With this setting, both `XTimer_SetInterval()` and `XTimer_SetHandler()` are **no-ops** — the tick timer is never configured and no interrupt fires.

**Fix:** Bypass xiltimer entirely and configure the ARM SCU Private Timer directly:

```c
// platform.c — SDT path
#include "xscutimer.h"

static XScuTimer ScuTimerInst;

static void ScuTimerIntrHandler(void *CallBackRef)
{
    XScuTimer_ClearInterruptStatus(&ScuTimerInst);
    timer_callback();
}

void init_timer()
{
    XScuTimer_Config *ConfigPtr =
        XScuTimer_LookupConfig(XPAR_SCUTIMER_BASEADDR);
    XScuTimer_CfgInitialize(&ScuTimerInst, ConfigPtr, ConfigPtr->BaseAddr);
    XScuTimer_EnableAutoReload(&ScuTimerInst);
    // SCU timer runs at CPU_CLK/2; divide by 20 for 50 ms
    XScuTimer_LoadTimer(&ScuTimerInst, XPAR_CPU_CORE_CLOCK_FREQ_HZ / 40);
    XScuTimer_EnableInterrupt(&ScuTimerInst);
    XScuTimer_Start(&ScuTimerInst);
    // Registers handler in GIC and enables CPU IRQ via Xil_ExceptionEnable()
    XSetupInterruptSystem(&ScuTimerInst,
                          (Xil_InterruptHandler)ScuTimerIntrHandler,
                          ScuTimerInst.Config.IntrId,
                          ScuTimerInst.Config.IntrParent,
                          XINTERRUPT_DEFAULT_PRIORITY);
}
```

---

### Note — UART Console Port

The ZYBO Z7-20 exposes **two USB COM ports** when connected to a PC. Only the **second** port is the UART console (`PS UART1`, `STDOUT_BASEADDRESS = 0xE0001000`). The first port is used for JTAG programming only.

Serial settings: **115200 baud, 8N1**.

---

### Note — Actual TCP Port

`echo.c`'s `print_app_header()` prints `"TCP packets sent to port 6001 will be echoed back"` — this is a **bug in the template**. The actual listening port set in `start_application()` is **port 7**.

---

## Customizing the Echo Logic

To replace the echo with your own logic, edit `recv_callback()` in [`Vitis/lwip_echo_server/src/echo.c`](Vitis/lwip_echo_server/src/echo.c):

```c
err_t recv_callback(void *arg, struct tcp_pcb *tpcb,
                    struct pbuf *p, err_t err)
{
    if (!p) { tcp_close(tpcb); return ERR_OK; }

    tcp_recved(tpcb, p->len);

    // ── Replace this block with your own logic ──────────────
    // p->payload : received data pointer
    // p->len     : received data length
    if (tcp_sndbuf(tpcb) > p->len)
        tcp_write(tpcb, p->payload, p->len, 1);  // echo
    // ────────────────────────────────────────────────────────

    pbuf_free(p);
    return ERR_OK;
}
```

---

## References

- [Digilent ZYBO Z7 Reference Manual](https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual)
- [Xilinx lwIP Library v2.2 (PG313)](https://docs.amd.com/r/en-US/pg313-embedded-ethernet)
- [RTL8211E-VL Datasheet](https://www.realtek.com/en/products/communications-network-ics/item/rtl8211e-vl-cg)
- [ARM Cortex-A9 SCU Timer TRM](https://developer.arm.com/documentation/ddi0407/latest)
- Full bring-up notes: [`troubleshooting_report.html`](troubleshooting_report.html)
