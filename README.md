# Mosey Extended — Pixel 8 Pro / husky Research

> [!WARNING]
> This is an experimental research project for a single, explicitly defined test platform:
> **Google Pixel 8 Pro (husky), Android 16, Build CP1A.260505.005.A1**.
>
> The project is not a claim of compatibility with Pixel 7/8/8a, Pixel 9/10, Samsung, BBK,
> or other devices. Results from other devices must be treated as separate research unless
> independently verified.
>
> This project is independent and is not affiliated with, authorized by, or endorsed by
> Google LLC or Apple Inc. AirDrop, Android, Google, Quick Share and related names are
> trademarks of their respective owners.

## Project goal

The objective is **real, bidirectional AirDrop file transfer between this Pixel 8 Pro and an iPhone**.

A successful result must include all of the following:

1. Pixel discovers iPhone / Apple devices.
2. iPhone discovers Pixel.
3. Required authentication / channel establishment succeeds.
4. A real data communication path is established.
5. Pixel → iPhone file transfer succeeds.
6. iPhone → Pixel file transfer succeeds.
7. The final behavior should approach the native Quick Share ↔ AirDrop experience as closely as practical.

The following are **not** considered success by themselves:

- Quick Share UI changes
- AirDrop / Quick Share icons appearing
- a feature flag being enabled
- BLE discovery alone
- creation of `wonder0`
- creation of a virtual PHY
- successful startup of `MoseyApp` or `mosey_server`
- successful registration of a vendor command
- existence of `radiotap0`

---

## 1. Fixed research platform

All conclusions in this repository are scoped to this exact platform unless explicitly stated otherwise.

| Item | Current value |
|---|---|
| Device | Google Pixel 8 Pro |
| Codename | `husky` |
| SoC | Tensor G3 |
| Android | Android 16 |
| Build | `CP1A.260505.005.A1` |
| SDK | `36` |
| Kernel | `6.1.145-android14-11-gfa1d6308d1fe-ab14691759` |
| Kernel toolchain | clang 17.0.2 / aarch64 |
| Kernel | Android GKI, SMP PREEMPT |
| SELinux | Enforcing |
| Boot slot | `_b` |
| Verified Boot | green |
| Root | KernelSU Next |
| WLAN driver | `bcmdhd4398` |

### Important correction: Wi-Fi chipset

Earlier project material incorrectly classified Pixel 8 / 8 Pro as BCM4389.

That classification is **not used by this project anymore**.

For the current target device, the hardware basis is:

```text
Pixel 8 Pro (husky)
    ↓
bcmdhd4398
    ↓
BCM4398
```

This is an **[EMPIRICAL / DEVICE-VERIFIED]** project fact and takes precedence over older README or PR descriptions.

---

## 2. Evidence policy

Every technical statement in this repository should be classified into one of four evidence levels.

### [EMPIRICAL]
Verified directly on the target Pixel 8 Pro.

Examples:

- running kernel version
- loaded `bcmdhd4398`
- existence and properties of `radiotap0`
- existence of `aware_nmi0`
- extracted module hashes / Build IDs
- actual process and logcat observations

### [SOURCE-CONFIRMED]
Supported by Google / Android / Broadcom source code or by direct source inspection of the project.

Examples:

- Google Wonder exists as a real kernel component
- BCM4398 DHD source contains monitor/radiotap support
- BCM4398 DHD source contains Action Frame TX/RX paths
- `aware_nmi0` is part of the NAN/NMI implementation
- Google Wonder uses a vendor-driver integration layer / WonderTap architecture

### [AUTHOR-CLAIM]
A claim made by a project author, issue, PR or README that has not been independently verified on this Pixel 8 Pro.

Examples:

- PR #8's claim of theoretical support
- claims that older Pixel devices can obtain usable AirDrop RF through `wonder_mosey_wild.ko`
- generic multi-device compatibility claims

### [INFERENCE]
A technical hypothesis derived from evidence but not yet experimentally proven.

Examples:

- `radiotap0` is the DHD monitor/radiotap netdev created by the BCM4398 stack
- `radiotap0` TX may ultimately reach BCM4398 firmware
- a WonderTap adapter may be sufficient if the existing monitor/RF path is usable

No inference should be promoted to a fact without a corresponding test or source proof.

---

## 3. Current kernel / WLAN facts

The target kernel has already been checked and is known to provide:

```text
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
CONFIG_MODULES=y
CONFIG_MODVERSIONS=y
CONFIG_WLAN=y
```

The running system has:

```text
mac80211
cfg80211
bcmdhd4398
```

Therefore this project is **not** starting from a kernel that lacks mac80211/cfg80211.

The active WLAN stack is at least:

```text
Pixel 8 Pro
    ↓
BCM4398
    ↓
bcmdhd4398
    ↓
mac80211 / cfg80211
```

---

## 4. Important discovered interfaces

### `radiotap0`

The target device exposes:

```text
radiotap0
```

Observed properties:

```text
ifindex      49
type         803 (ARPHRD_IEEE80211_RADIOTAP)
iflink       49
state        DOWN
operstate    down
address      00:00:00:00:00:00
rx_packets   0
tx_packets   0
```

The sysfs path is:

```text
/sys/devices/virtual/net/radiotap0
```

There is no `wireless/` directory and no `phy80211` entry.

**[EMPIRICAL]** `radiotap0` is therefore a virtual Linux netdev with IEEE 802.11 radiotap type 803.

**[SOURCE-CONFIRMED]** BCM DHD monitor implementations use a virtual radiotap netdev and contain monitor TX/RX handling.

**[INFERENCE]** The current `radiotap0` is very likely associated with the BCM4398 DHD monitor subsystem.

What is **not yet proven**:

```text
radiotap0 TX
    ↓
bcmdhd4398
    ↓
BCM4398 firmware
    ↓
RF
```

This exact path is one of the primary remaining research targets.

### `aware_nmi0`

Observed:

```text
ifindex      46
type         1
state        DOWN
address      00:90:4c:33:22:11
```

**[SOURCE-CONFIRMED]** Google/Broadcom BCM DHD NAN source defines `aware_nmi0` as the NMI interface used by the NAN implementation.

The NAN NMI transmit callback is not a normal data-plane TX path; the source explicitly treats the interface as an auxiliary/control mechanism.

Therefore:

```text
aware_nmi0 ≠ generic raw 802.11 data TX interface
```

It remains relevant to NAN / discovery research, but it should not be treated as the final AirDrop RF transport.

---

## 5. BCM4398 driver binary investigation

Two stock modules exist on the target device:

```text
/vendor_dlkm/lib/modules/bcmdhd4398.ko
/vendor_dlkm/lib/modules/16k-mode/bcmdhd4398.ko
```

They are **not identical binaries**.

### Standard module

```text
SHA-256:
e2c3656e56bf94763e00a51182c16ca0db472282206323410177075a4c8751dd

Size:
9,921,072 bytes

Build ID:
de704232a38f39a56f6501d0d9fce3e3f15abf92
```

### 16K module

```text
SHA-256:
59a6588bf23855f7903d715d5071967f48b516cb65ecc4f3ed80b7c9c4ada8d4

Size:
9,929,264 bytes

Build ID:
05757726fed97d4195a55874b5348095a6ab635c
```

Both are reported by `file` as:

```text
ELF 64-bit LSB relocatable, ARM aarch64, not stripped
```

The running kernel exposes the loaded module as:

```text
/sys/module/bcmdhd4398
```

and reports:

```text
scmversion = g64bfcee46f4f
coresize   = 3870720
initstate  = live
```

The module file `vermagic` previously observed is:

```text
6.1.145-android14-11-g164ae0b804dd-ab15037554
```

> Do not use the abbreviated line above for ABI decisions. The exact module metadata and running-kernel relationship still require dedicated investigation.

### Why the `.ko` is especially valuable

The extracted module is **not stripped** and contains `.symtab` / `.strtab` in the extracted ELF. This makes function-level static analysis practical.

The next analysis target is **not a newly compiled module**.

It is the stock binary:

```text
bcmdhd4398.ko
```

For the first pass, use the standard 4K-tree copy:

```text
/Users/patrick/Downloads/bcmdhd4398.ko
```

The separate 16K variant should only be analyzed afterward if the first binary does not explain the running path.

---

## 6. What the stock BCM4398 binary already proves

The extracted `bcmdhd4398.ko` contains visible symbols / strings associated with all of the following:

### Monitor / radiotap

```text
dhd_add_monitor_if
dhd_del_monitor_if
dhd_monitor_open
dhd_monitor_stop
dhd_monitor_ioctl
dhd_set_monitor_ioctl
dhd_monitor_enabled
wl_cfg80211_add_monitor_if
wl_cfg80211_set_monitor_channel
radiotap
DHD-MON
```

### Action Frame TX/RX

```text
wl_cfg80211_send_action_frame
wl_cfgp2p_tx_action_frame
wl_cfg80211_abort_action_frame
wl_cfg80211_actframe_fillup_v2
wl_cfgp2p_action_tx_complete
wl_notify_rx_mgmt_frame
WLC_E_ACTION_FRAME_RX
WLC_E_ACTION_FRAME_COMPLETE
```

### Data TX/RX

```text
dhd_prot_txdata
dhd_bus_txdata
dhd_rx_frame
dhd_bus_rx_frame
```

### NAN

The binary contains extensive NAN implementation symbols and event strings, including:

```text
wl_cfgnan_init
wl_cfgnan_attach
wl_cfgnan_start_handler
wl_cfgnan_transmit_handler
wl_cfgvendor_nan_transmit
wl_cfgvendor_nan_data_path_iface_create
wl_cfgvendor_nan_data_path_iface_delete
wl_cfgnan_get_capablities
...
```

and explicit references to `aware_nmi0`.

### Important conclusion

**[SOURCE-CONFIRMED / BINARY-CONFIRMED]** The stock BCM4398 driver contains substantially more functionality than a basic Android Wi-Fi data driver. It has monitor/radiotap, Action Frame, NAN and low-level DHD TX/RX infrastructure.

This does **not** yet prove that the driver can perform arbitrary raw 802.11 frame injection suitable for Wonder/AirDrop.

---

## 7. WonderTap / Google Wonder status

Google Wonder is a real Google kernel component. It is not a concept invented by this project.

The generic Wonder architecture contains a virtual mac80211/SoftMAC layer and a vendor-driver integration mechanism commonly referred to as **WonderTap**.

The critical unresolved question for this device is:

```text
Does stock bcmdhd4398 provide a WonderTap-equivalent provider,
OR can one be attached to its existing monitor/RF path without
reimplementing the Wi-Fi radio?
```

### Current evidence

**[SOURCE-CONFIRMED]** Google Wonder uses a vendor-driver integration layer.

**[BINARY-CONFIRMED]** `bcmdhd4398.ko` has substantial monitor/radiotap and Action Frame infrastructure.

**[BINARY NEGATIVE EVIDENCE]** Straight `strings` searches of the extracted module have not exposed an obvious `wondertap`, `wonder0`, or similar Wonder provider name.

**[NOT PROVEN]** Absence from `strings` is not proof that no equivalent provider exists.

Therefore the current state is:

```text
bcmdhd4398 → monitor/radiotap        CONFIRMED
bcmdhd4398 → real Action Frame TX    CONFIRMED
bcmdhd4398 → real Action Frame RX    CONFIRMED
bcmdhd4398 → NAN/Aware               CONFIRMED
bcmdhd4398 → WonderTap               UNKNOWN
radiotap0 → firmware TX/RX            UNKNOWN
```

---

## 8. Why `wonder_mosey_wild.ko` is not the current solution

PR #8 (`BasGame1/mosey-p8a`) is open and unmerged. Its own description calls the device support theoretical / untested.

The proposed `wonder_mosey_wild.ko` is a virtual mac80211 driver.

Its current implementation:

- registers a virtual `ieee80211_hw`
- exposes monitor/station interface modes
- registers vendor commands
- has vendor command handlers that largely log and return success
- frees transmitted sk_buffs in its TX callback
- does not implement a real RF hardware TX/RX backend

Therefore it must currently be classified as:

```text
virtual PHY / protocol-plumbing stub
```

and not as a complete AirDrop transport.

**Do not treat `wonder_mosey_wild.ko` as the final answer for Pixel 8 Pro.**

---

## 9. Current research architecture

The most useful current model is:

```text
                           Quick Share / Mosey
                                  │
                             mosey_server
                                  │
                             Wonder protocol
                                  │
                         generic Wonder / SoftMAC
                                  │
                     ┌────────────┴────────────┐
                     │                         │
                WonderTap?               radiotap path?
                     │                         │
                     └────────────┬────────────┘
                                  │
                            bcmdhd4398
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                 wlan0         NAN/Aware      monitor
                    │          aware_nmi0     radiotap0
                    │             │             │
                    └─────────────┴──────┬──────┘
                                         │
                                  BCM4398 firmware
                                         │
                                         RF
```

The dashed/unknown portions are intentional: they are research targets, not established facts.

---

## 10. Priority research questions

### P0 — `radiotap0` TX/RX path

Determine exactly how the registered `radiotap0` netdev connects to the DHD monitor implementation.

The target chain is:

```text
radiotap0
  ↓
netdev_ops / ndo_start_xmit
  ↓
DHD-MON TX handler
  ↓
DHD TX path
  ↓
BCM4398 firmware
```

For RX, establish the reverse path.

This is the highest-value question because a positive result would establish a realistic existing RF backend for a future Wonder adapter.

### P1 — Native Wonder provider

Search the extracted `bcmdhd4398.ko` for:

```text
wondertap
auxiliary_device
auxiliary_driver
wondertap_ops
wondertap_aux_dev
```

and inspect symbol relationships rather than relying only on strings.

### P2 — Action Frame capability limits

Determine whether the existing cfg80211 Action Frame path can carry the exact 802.11 management/action frames required by the Wonder/AirDrop implementation, including channel, dwell-time and firmware restrictions.

### P3 — Firmware / iovar interface

Map the monitor and action-frame functions to the underlying Broadcom ioctl/iovar interface and identify what the firmware actually accepts.

### P4 — Mosey ↔ lower transport integration

Only after the RF path is understood should `mosey_server` / Phenotype / service integration be modified.

### P5 — AirDrop wire protocol

In parallel, continue protocol work on:

- discovery
- authentication
- channel establishment
- transport/session setup
- file transfer

Protocol progress must remain separate from claims of transport success.

---

## 11. `nlmon0` and packet capture

`nlmon0` is a high-value diagnostic tool for a **specific** Mosey / Quick Share transaction.

It can be used to capture and compare:

- nl80211 operations
- vendor commands
- `NL80211_CMD_FRAME`
- NAN operations
- channel changes

Do not run long-lived captures without a concrete test scenario.

The highest-value comparison remains:

```text
working native platform
        vs.
Pixel 8 Pro implementation
```

with particular attention to the complete control-plane and transport sequence.

---

## 12. Development and safety policy

Until the hardware path is understood:

- prefer read-only inspection
- use `/data/local/tmp` for temporary files
- prefer temporary `insmod` only after ABI validation
- use KernelSU modules for reversible integration
- avoid direct boot / vendor_boot modifications
- avoid persistent vendor partition modification
- never auto-load an unverified `.ko`

### `.ko` loading requirements

No kernel module should be loaded merely because its filename or reported target version looks correct.

Before any load decision, verify:

1. exact running kernel version
2. module `vermagic`
3. relevant kernel configuration
4. required exported symbols / symbol versions
5. KCFI / MODVERSIONS compatibility where applicable
6. ABI assumptions in the module
7. rollback method

An unverified kernel module can cause driver failure, kernel panic or a bootloop.

---

## 13. Current milestone status

| Milestone | Status |
|---|---|
| Understand target device | ✅ |
| Confirm BCM4398 / bcmdhd4398 | ✅ |
| Confirm `radiotap0` exists | ✅ |
| Confirm `aware_nmi0` exists | ✅ |
| Confirm BCM4398 monitor/radiotap code | ✅ |
| Confirm BCM4398 Action Frame TX/RX code | ✅ |
| Confirm BCM4398 NAN implementation | ✅ |
| Prove `radiotap0` reaches firmware TX | ⏳ |
| Prove `radiotap0` receives firmware RX | ⏳ |
| Prove WonderTap/native Wonder provider | ⏳ |
| Connect generic Wonder to BCM4398 | ⏳ |
| Mosey discovery with real transport | ⏳ |
| Pixel ↔ iPhone authentication/channel | ⏳ |
| Pixel → iPhone file transfer | ⏳ |
| iPhone → Pixel file transfer | ⏳ |
| Native-like user experience | ⏳ |

---

## 14. Next binary-analysis target

The next file to analyze is the **stock driver binary already extracted from the device**:

```text
bcmdhd4398.ko
```

Use this exact local file for the first pass:

```text
/Users/patrick/Downloads/bcmdhd4398.ko
```

Do **not** substitute `wonder_mosey_wild.ko` for this investigation.

Do **not** start by modifying the binary.

The purpose of the next analysis is to recover the symbol-level path for:

```text
radiotap0
  → monitor netdev ops
  → TX/RX callbacks
  → DHD monitor functions
  → DHD protocol/bus TX/RX
```

The 16K variant is a separate binary and should be analyzed only after the standard binary path has been mapped.

---

## 15. Primary research references

- Google Android common Wonder source: `drivers/android/wonder/`
- Google WonderTap interface: `include/linux/android/wondertap.h`
- Google BCM4398 source tree: `kernel/google-modules/wlan/bcmdhd/bcm4398`
- `mosey-extended` upstream: `thelok1s/mosey-extended`
- PR #8: `BasGame1/mosey-p8a`

This README is intentionally a **research log and engineering specification**, not a compatibility claim.

When new evidence conflicts with an earlier conclusion, update the conclusion explicitly and record what changed.
