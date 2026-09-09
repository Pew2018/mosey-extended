# Mosey Extended — Pixel 8 Pro / husky Research

> [!WARNING]
> Experimental research project targeting **Google Pixel 8 Pro (husky), Android 16**. This project is independent and is not affiliated with Google or Apple.

## Goal

Achieve **real bidirectional AirDrop file transfer between a Pixel 8 Pro and iPhone**, using Root/KernelSU Next, Mosey/Wonder, and the device's existing Wi-Fi stack.

Success requires:

- Pixel ↔ iPhone discovery
- authentication / channel establishment
- real 802.11/IP data communication
- Pixel → iPhone file transfer
- iPhone → Pixel file transfer
- preferably an experience close to native Quick Share ↔ AirDrop

UI changes, feature flags, BLE discovery alone, `wonder0`, `radiotap0`, a virtual PHY, or a running `mosey_server` are **not** considered AirDrop success.

## Target platform

| Item | Value |
|---|---|
| Device | Google Pixel 8 Pro |
| Codename | `husky` |
| SoC | Tensor G3 |
| Android | 16 |
| Build | `CP1A.260505.005.A1` |
| SDK | 36 |
| Kernel | `6.1.145-android14-11-gfa1d6308d1fe-ab14691759` |
| WLAN | `bcmdhd4398` / **BCM4398** |
| Root | KernelSU Next |
| SELinux | Enforcing |

### Important hardware correction

Older upstream material associated Pixel 8/8 Pro with BCM4389. **That is not the hardware basis used by this research.** Direct investigation of the target device identifies the active WLAN stack as **BCM4398 / `bcmdhd4398`**. Device evidence takes precedence over older project descriptions.

## Current progress

### Confirmed

- Pixel 8 Pro hardware and software platform identified.
- Active `bcmdhd4398` driver identified.
- `mac80211`, `cfg80211` and the Broadcom DHD stack are present.
- `radiotap0` exists as an `ARPHRD_IEEE80211_RADIOTAP` (type 803) virtual netdev.
- `aware_nmi0` exists and is associated with the NAN/NMI subsystem; it is **not** treated as a generic raw 802.11 data interface.
- Stock `bcmdhd4398.ko` is available for static analysis and retains symbols/relocations.
- The driver contains monitor/radiotap, Action Frame, NAN and DHD TX/RX infrastructure.
- `dhd_add_monitor_if()` creates/registers a type-803 monitor netdev.
- `dhd_mon_if_subif_start_xmit()` performs real skb handling and calls `dhd_start_xmit()`.
- The analyzed TX path continues through:

```text
dhd_mon_if_subif_start_xmit()
        ↓
dhd_start_xmit()
        ↓
dhd_lb_sendpkt()
        ↓
__dhd_sendpkt()
        ↓
dhd_flowid_update()
        ↓
dhd_prot_hdrpush()
        ↓
dhd_bus_txdata()
```

- The complete `__dhd_sendpkt()` analysis directly shows the call to `dhd_bus_txdata()`.

### Not yet proven

- `radiotap0`'s exact `ndo_start_xmit` mapping to `dhd_mon_if_subif_start_xmit()`.
- `dhd_bus_txdata()` → TX ring / msgbuf → DMA / PCIe → firmware → **actual RF transmission**.
- Actual RF TX and RX from `radiotap0`.
- A native WonderTap provider in the BCM4398 driver.
- Complete Mosey userspace integration on this Pixel 8 Pro.
- iPhone/Pixel AirDrop discovery in both directions.
- AirDrop authentication and session establishment.
- Pixel → iPhone transfer.
- iPhone → Pixel transfer.
- A final production-ready KernelSU module.

## Current architecture / strategy

The central research question is how to connect the already available **Wonder/Mosey data path** to the **real BCM4398 transport** without unnecessarily reimplementing the Wi-Fi driver.

### Route A — reuse the stock BCM4398 monitor/radiotap path

**Current preferred route:**

```text
Mosey / Wonder
      ↓
transport bridge
      ↓
radiotap / DHD monitor path
      ↓
stock bcmdhd4398
      ↓
BCM4398 firmware
      ↓
RF
```

This route is attractive because the stock driver already contains a substantial monitor TX/RX path. The key remaining task is to prove that path reaches real hardware and then expose it cleanly to Wonder/Mosey.

### Route B — implement a real WonderTap provider

Investigate whether BCM4398 already contains a WonderTap-equivalent integration or whether a small adapter can connect Google Wonder to the existing driver.

This is architecturally attractive and closer to Google's Wonder design, but currently **unproven on BCM4398**.

### Route C — patch the stock BCM4398 driver

Possible if Routes A/B cannot provide the required interface. This is a higher-risk option because of Android GKI, KCFI, `CONFIG_MODVERSIONS` and driver ABI concerns.

### Route D — `wonder_mosey_wild.ko`

Useful as a **virtual PHY / protocol-plumbing prototype** for testing Wonder/Mosey integration. It is not currently considered a real RF backend and therefore is not the final solution by itself.

## Upstream contributions

The upstream `mosey-extended` work provides important upper-layer foundations, including:

- `mosey_server` research and integration
- Wonder / `wonder0` architecture research
- nl80211 / cfg80211 vendor-command understanding
- virtual Wonder PHY prototype
- KernelSU/Magisk module structure
- init and SELinux integration work
- Quick Share / feature-flag research
- AirDrop / Quick Share capture and protocol-analysis methodology

The capture methodology is particularly useful for establishing a **real iPhone ↔ supported-device wire-level reference** before attempting to reproduce the protocol on Pixel 8 Pro.

Upstream README/PR statements are treated as project claims unless independently verified on this target device.

## Next milestones

1. Prove `radiotap0` → DHD monitor TX callback mapping.
2. Trace `dhd_bus_txdata()` down to the actual TX ring / msgbuf / DMA / PCIe / firmware path.
3. Determine monitor frame format and firmware requirements.
4. Perform the first externally verified real 802.11 RF TX test.
5. Verify RF RX.
6. Connect Wonder/Mosey packet I/O to the working BCM4398 transport.
7. Stabilize `mosey_server` and related Android integration.
8. Establish AirDrop discovery and authentication.
9. Achieve Pixel → iPhone transfer.
10. Achieve iPhone → Pixel transfer.
11. Package the verified implementation as a reversible KernelSU module.

## Evidence policy

Research conclusions are classified as:

- **[EMPIRICAL]** — directly verified on the target Pixel 8 Pro.
- **[SOURCE-CONFIRMED]** — supported by source code or direct binary analysis.
- **[AUTHOR-CLAIM]** — stated by an upstream/project author but not independently verified here.
- **[INFERENCE]** — a hypothesis derived from available evidence.

A kernel function reaching `dhd_bus_txdata()` is **not** by itself proof of RF transmission. A `radiotap0` or `wonder0` interface is **not** proof of AirDrop. Only externally observable RF/data exchange and successful bidirectional file transfer can close those gaps.

## Status

**Research phase — real AirDrop transfer is not yet achieved.**

The project has progressed from investigating a hypothetical virtual interface toward a concrete, device-specific BCM4398 transport investigation. The immediate focus is the stock DHD TX/RX path and its eventual connection to Wonder/Mosey.