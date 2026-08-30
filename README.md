# bsp-mic-743

Board support for the **Advantech MIC-743** — the NVIDIA Jetson AGX Thor **T5000**
(P3834-0008 CVM) variant of Advantech's MIC-74x industrial edge AI system.

Layers on top of the `jetson-agx-thor` target. 2026 feed only.

The **MIC-742** is the same carrier with a T4000 (P3834-0000) module and has its own
extension, [`bsp-mic-742`](https://github.com/avocado-linux/bsp-mic-742). The carrier
half of the two is identical; only the module flashvars differ.

## What this extension carries

Advantech's Flash SOP runs the **stock** NVIDIA profile
(`l4t_initrd_flash.sh jetson-agx-thor-devkit internal`), so the `avocado-jetson-agx-thor`
MACHINE defaults are already the right baseline. This extension carries only the carrier
delta, derived by diffing Advantech's `MIC-742_Thor_7.2_V1.0.0_SDK` against the stock
NVIDIA `Jetson_Linux_R39.2.0_aarch64` BSP — **the same L4T release wrynose pins**, so
nothing here is a cross-release backport.

### Flash-time (`stone/carrier-bsp/`)

| File | What it does |
|---|---|
| `tegra264-mb1-bct-pinmux-p3834-xxxx-p4071-0000.dtsi` | 61 changed pin blocks: CAN2/CAN3, eth0/eth3 MDIO+MDC, gen3/gen9 I2C, PWM2/3, DAP2/DAP6 audio |
| `tegra264-mb1-bct-gpio-p3834-xxxx-p4071-0000.dtsi` | Carrier boot GPIO defaults — pin-header buffers and direction pins, internal USB VBUS enable, AON DI/DO inputs |
| `tegra264-mb1-bct-prod-p3834-xxxx-p4071-0000.dts` | I2C1 drive-strength fix so MB1 can read the SoM EEPROM through the carrier's 1.8↔3.3V level shifter (NVIDIA bug 5459342) |
| `tegra264-bpmp-3834-0008-4071-xxxx-adv.dtb` | Advantech's BPMP DTB — `pcie@3` enabled |
| `tegra264-mic-74x-carrier.dtbo` | The kernel-DT delta, as an overlay |
| `carrier.env` | `CARRIER_FV_*` / `CARRIER_ENV_*` knobs |

The first three ship under their **stock filenames** and shadow the `tegraflash-bsp`
copies. That is deliberate: meta-tegra generates thin `.dts` wrappers that `#include` the
pinmux/gpio `.dtsi` *by name*, so renaming them with an `-adv` suffix (the Orin-era
convention) would orphan the include.

### The carrier overlay

The entire kernel-DT difference between the stock AGX Thor devkit DTB and the DTB
Advantech ships is **four fragments**, and it is byte-identical between the T5000 and
T4000 DTBs — which is why this is a 1KB overlay shared by both boards rather than two
forked 256KB DTBs:

1. **`/bus@0/spi@810c440000/spi@0`** — retargets SPI2 CS0 from generic `tegra-spidev`
   to `infineon,slb9670` at 1MHz: the discrete TPM 2.0.
2. **`/bus@0/pcie@a808440000`** — enables PCIe C3, the ASMedia ASM106x SATA controller.
3. **`/dce@8808000000/sha-carveout`** — enabled.
4. **`/sound`** — adds the internal-speaker widget and LOUTL/LOUTR routing on the RT5640.

Enabling SATA needs all three of: `CARRIER_ENV_ODMDATA` (MB1 powers and maps the C3 UPHY
lanes), `pcie@3` in the carrier BPMP DTB, and fragment 2 above. Advantech sets all three.

**Verification.** The overlay was validated offline: applied to the stock R39.2.0
`tegra264-p4071-0000+p3834-0008-nv.dtb` with `fdtoverlay`, the result reproduces
Advantech's shipped DTB exactly for all four fragments.

```sh
dtc -I dts -O dtb -o tegra264-mic-74x-carrier.dtbo tegra264-mic-74x-carrier.dts
fdtoverlay -i <stock>/tegra264-p4071-0000+p3834-0008-nv.dtb \
           -o applied.dtb tegra264-mic-74x-carrier.dtbo
# diff applied.dtb against Advantech's DTB -> only the documented usb3-1 delta
```

One Advantech change is **deliberately not carried**: they also disable the `usb3-1` lane
and port and trim it out of the `phys`/`phy-names` arrays on the xhci and xudc nodes,
because the carrier wires only usb3-0 and usb3-2. Rewriting those arrays from an overlay
means re-emitting a phandle list, and the base DTB exposes no `__symbols__` labels for the
individual padctl lanes. Disabling the lane *without* trimming the arrays would be worse
than leaving it alone — xhci-tegra would still resolve a phy for a disabled lane on a
boot-critical controller. Left enabled, usb3-1 is an unwired port that never links,
exactly as on the devkit.

## TPM: discrete, not fTPM

The MIC-74x carries an **Infineon SLB9670 TPM 2.0** on SPI2 CS0. Confirmed on hardware:
`spi2.0` with modalias `spi:slb9670`, `/dev/tpm0` present, TPM major version 2.

This extension therefore **replaces** the MACHINE's `OVERLAY_DTB_FILE`
(`tegra264-ftpm.dtbo`) rather than appending to it, and ships
`kernel-module-tpm-tis-spi` but **not** `kernel-module-tpm-ftpm-tee`.

Shipping both would give Linux two TPMs — the OP-TEE fTPM and the discrete part — racing
for `/dev/tpm0` vs `/dev/tpm1` with no deterministic probe order. Anything sealing to a
PCR (`cryptsetup-var` seals to PCR 7) would silently bind to whichever appeared first.

> A package-install choice alone **cannot** express this: the device node only exists if
> the DT node does, and the DT node is decided at flash time. Offering users a
> fTPM-vs-discrete choice means shipping a second `.dtbo` and selecting it in
> `carrier.env`, not just installing a different module.

The discrete part is the better default here: it is an independent hardware root of trust
that survives a compromise of the SoC's trusted OS, whereas the fTPM is fate-shared with
it. The trade-off is that a discrete TPM sits on a sniffable/interposable SPI bus, so use
TPM2 salted + parameter-encrypted sessions for anything sensitive.

## Carrier hardware

Confirmed on a running MIC-743:

- Infineon SLB9670 TPM 2.0 (`spi2.0`, `/dev/tpm0`)
- ASMedia ASM106x SATA AHCI on PCIe C3 (`0003:01:00.0`)
- 4× on-SoC MGBE 10GbE (`nvethernet`) + Realtek r8126
- 4× CAN (`mttcan`), 4× `ttyS`
- RT5640 audio, AT24 EEPROMs, TMP451/LM90, INA3221/INA238
- M.2 E-key (WiFi/BT) and B-key (WWAN) option slots

SATA needs no kernel modules — this kernel has `CONFIG_ATA=y`, `CONFIG_SATA_AHCI=y`,
`CONFIG_SCSI=y` and `CONFIG_BLK_DEV_SD=y` built in.

> **WWAN note.** The QMI path (ModemManager's preferred one) needs
> `CONFIG_USB_NET_QMI_WWAN`, `CONFIG_USB_WDM` and `CONFIG_USB_SERIAL_QUALCOMM`, all
> currently unset in the wrynose Tegra kernel. Only the AT/PPP path
> (`usbserial` → `usb-wwan` → `option`) works today. Enabling QMI is a 3-line config
> fragment in `meta-avocado`, not something this extension can fix.

## Using this extension

`bsp-mic-743` is an [Avocado](https://avocadolinux.org) extension — a reusable fragment of
build- and runtime-configuration that you compose into your own Avocado project. To use it,
declare it as a package-sourced extension in your `avocado.yaml` and add it to a runtime:

```yaml
extensions:
  avocado-bsp-mic-743:
    source:
      type: package
      version: "*"        # or pin an exact version

runtimes:
  my-runtime:
    extensions:
      - avocado-bsp-mic-743
```

Then install and build:

```sh
avocado install   # fetches + installs the SDK, extensions and runtime deps from your config
avocado build     # builds the SDK compile steps, extensions and runtime images
```

`avocado install` pulls the extension from your target's package feed and merges its
config into your project; `avocado build` then produces the runtime.
