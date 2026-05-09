# tc8-kernel-patches

Mainline Linux 6.6 patches for the Polycom TC8 video conferencing
panel (i.MX 8M Mini, codename **LCC**). Enables CPU, eMMC, networking,
audio, panel, and touch on a vanilla upstream tree.

## Apply

```bash
git clone --branch v6.6 --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git am ../tc8-kernel-patches/patches/*.patch
```

Applies cleanly with `git am` against `v6.6`.

## Patches

| # | Patch | Subsystem |
|---|---|---|
| 0001 | `arm64: dts: freescale: add imx8mm-tc8 board support` | arm64 / DT |
| 0002 | `drm/panel: add Polycom TC8 LCC MIPI-DSI panel driver` | drm/panel |
| 0003 | `net: dsa: realtek: add RTL8363NB-VB switch support` | net/dsa |
| 0004 | `net: fec: support fixed-phy DSA conduit on TC8` | net/fec |
| 0005 | `ASoC: tas571x: add TAS5751M support` | ASoC |

### 0001 — `arm64: dts: freescale: add imx8mm-tc8 board support`

New device tree `arch/arm64/boot/dts/freescale/imx8mm-tc8.dts`.
Describes:

- i.MX 8M Mini quad Cortex-A53 + 16 GB eMMC
- FEC1 RGMII fixed-link to on-board RTL8363NB-VB DSA switch
- TAS5751M on I²C2 + SAI1 (24-bit I²S, MCLK output)
- Goodix GT9110 multi-touch on I²C2 (compatible `goodix,gt9271`,
  reset-gpios on gpio2.3 active-low)
- `lcc,proto` MIPI-DSI panel (800×1280, 4 lanes), bound to driver in
  patch 0002
- Pin-mux + clocks for FEC1 RGMII, SAI1, USDHC2, panel/touch reset

Board file only; no driver added or modified.

### 0002 — `drm/panel: add Polycom TC8 LCC MIPI-DSI panel driver`

New `drivers/gpu/drm/panel/panel-poly-lcc.c`, registered as
`raydium,poly-lcc` (compatible `lcc,proto`). Forked from
`panel-raydium-rm68200.c`; reuses upstream Raydium MCS register
defines and panel funcs structure.

- Mode: 800×1280 @ ~60 Hz, 76.556 MHz pixel clock
- 4 DSI lanes, video mode, sync-pulse + LPM + non-continuous clock
- Reset-gpios held deasserted across prepare/enable
- DCS panel-init body is empty (no manufacturer commands sent);
  prepare/enable issues only `exit_sleep_mode` and `set_display_on`

Display rotation is not handled in the driver.

### 0003 — `net: dsa: realtek: add RTL8363NB-VB switch support`

Adds RTL8363NB-VB chip support (chip ID `0x6511`) to
`drivers/net/dsa/realtek/rtl8365mb.c`:

- New `chip_info` entry referencing the TC8 init jam table
- 63-entry SoC-side register replay (precondition for PHY access)
- 1507-byte 8051 microcode loader, fetched via `request_firmware()`
  from `/lib/firmware/rtl8363nbvb-8051.bin`
- 210-entry PHY OCP patch table sequenced after firmware upload
- Uses `DSA_TAG_PROTO_RTL8_4` (CPU egress is dropped without the
  tag)

Surfaces one user port `lan` + CPU port to mainline DSA.

### 0004 — `net: fec: support fixed-phy DSA conduit on TC8`

Two changes to `drivers/net/ethernet/freescale/fec_main.c`:

- Adds `FEC_QUIRK_NO_HARD_RESET` to `fec_imx6q_info` and forces
  `fec_restart()` to use the soft-disable path unconditionally.
  Hard-resetting the FEC IP on TC8 silently kills its MDIO.
- In `fec_enet_mii_probe()`, when no PHY device is registered on
  the bus, registers a fixed-phy on the fly (link=1, 1 Gb/s, full
  duplex) and connects to it directly. Lets DSA conduit come up
  without a `fixed-link` subnode in the FEC DT node.

### 0005 — `ASoC: tas571x: add TAS5751M support`

Adds `tas5751_chip` to `sound/soc/codecs/tas571x.c`:

- Adds `int (*init)(struct tas571x_private *)` callback to
  `struct tas571x_chip`; called at probe after OSC_TRIM
- Reuses `tas5717_chip` regmap layout
- `vol_reg_size = 2` (TAS5721's default 1 truncates the master
  volume MSB)
- `tas5751_init()` issues 29 register writes covering SYS_CTRL_1,
  SDI, master/channel volumes, modulation limit, inter-channel
  delays, BD-mode INPUT_MUX, board-specific PWM_OUT_MUX (0x25 =
  0x01002245), and DRC unity coefficients
- Adds `ti,tas5751` of_match + i2c_id entries

## Licensing

Patches are GPL-2.0-only as derivative work of GPL-2.0-only kernel
source. See each patch header for `Signed-off-by:` and the modified
files for full licensing context.
