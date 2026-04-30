# DCENT_axe

> AI-native open-source firmware for the BitAxe family, built by
> [D-Central Technologies](https://d-central.tech/).
> A clean-room Rust rewrite of BitAxe firmware with the world's first
> built-in **MCP (Model Context Protocol) server** — every Bitaxe becomes
> an AI-controllable mining device and smart space heater.

**DCENT_axe** is the Bitaxe-class sibling to
[DCENT_OS](https://github.com/DCentralTech/DCENT_OS). It is designed for the same
home-miner audience, with the same philosophy — *quiet by default,
transparent, locally controlled, no cloud account, no mandatory devfee* —
but on the ESP32-S3 + open-hardware Bitaxe boards instead of industrial
Antminers.

It is a **clean-room rewrite in Rust**, informed by the public
[ESP-Miner](https://github.com/bitaxeorg/ESP-Miner) project as a protocol
reference, but not a fork: every line of `dcentaxe` is D-Central's own,
GPL-3.0 from day one.

---

## Why this exists

The Bitaxe is the most exciting open-hardware Bitcoin product of the last
decade. The default firmware (ESP-Miner) is great, and the ecosystem around
it is healthy. So why build another firmware?

Three reasons:

1. **Memory-safe Rust on ESP-IDF v5.3+.** The BitAxe runs on an ESP32-S3
   with ~300 KB of internal heap. Rust's ownership model plus a clean-room
   architecture make it possible to fix entire bug *classes* (use-after-free
   on shared work objects, partial-frame UART corruption, allocator
   fragmentation under sustained mining) instead of one report at a time.
2. **AI-native control, by design.** DCENT_axe ships an
   [MCP](https://modelcontextprotocol.io/) server over JSON-RPC 2.0 with
   12+ tools and 3 live resources, so any MCP-aware AI client (Claude,
   Cursor, custom agents, your own scripts) can read miner state and drive
   it directly. Set frequency, swap pools, run an autotune, query the
   block-tile, fetch history — over a documented protocol, not a scraped
   HTML form.

DCENT_axe stays neutral toward existing pools and existing AxeOS workflows.
It is not pool-locked, not vendor-locked, and not D-Central-locked.

---

## How it works

```
+---------------+    +-----------------------------------+    +---------------+
|  Bitcoin pool | <- |  DCENT_axe (Rust on ESP-IDF v5.3) | -> |  BM1366 /     |
|  (any V1/V2)  |    |  - dispatcher (work + slot scan)  |    |  BM1368 /     |
+---------------+    |  - Stratum V1 / V2 client         |    |  BM1370 /     |
                     |  - autotuner (Hz / W / J/TH / T)  |    |  BM1397       |
+---------------+    |  - power: TPS546 + DS4432U        |    |  ASICs over   |
|  AI client    | -> |  - thermal: EMC2101 / EMC2103     |    |  UART daisy   |
|  (Claude etc.)|    |  - MCP JSON-RPC 2.0 server        |    |  chain        |
|  via MCP      |    |  - Home Assistant MQTT bridge     |    +---------------+
+---------------+    +------------------+----------------+
                                        |
                                        | local LAN (HTTP + MQTT + MCP)
                                        v
                  +------------------------------------------+
                  |  Web dashboard (cyberpunk terminal v8)   |
                  |  + Tamagotchi OLED carousel (8 pages)    |
                  +------------------------------------------+
```

The firmware is a multi-crate Rust workspace targeting `xtensa-esp32s3-espidf`:

- `dcentaxe/` — main binary + embedded web dashboard.
- `dcentaxe-asic/` — BM1366 / BM1368 / BM1370 / BM1397 drivers, CRC, PLL,
  and the `AsicDriver` trait.
- `dcentaxe-hal/` — board detection, UART, I²C, TPS546 power, fan PWM, GPIO,
  EMC2101/EMC2103/TPS546 temperature.
- `dcentaxe-mining/` — work dispatcher, rolling hashrate, share tracking.
- `dcentaxe-stratum/` — Stratum V1 client (shared with DCENT_OS).
- `dcentaxe-stratum-v2/` — Stratum V2 implementation.
- `dcentaxe-bap/` — BAP UART accessory protocol (compatible with Bitaxe
  Touch–style displays and the [DCENT_ExpansionPack](https://github.com/DCentralTech/DCENT_ExpansionPack)
  BAP header).
- `dcentaxe-design-bundle/` — design system tokens shared with DCENT_OS.

---

## Key features

- **All current Bitaxe variants supported** — Max (BM1397), Ultra
  (BM1366), Supra (BM1368), Gamma / GT (BM1370), Hex Ultra (6× BM1366),
  Hex Supra (6× BM1368).
- **Rust on ESP-IDF v5.3+.** Memory-safe core, alloc-free panic + OOM
  hooks, NVS breadcrumb on crash for post-mortem.
- **PSRAM enabled.** +8 MB of heap on supported S3 modules — eliminates
  the recurring OOM cycle that plagues memory-tight BitAxe firmwares.
- **Robust UART recovery.** Fallback slot scan + frame-recovery clear on
  partial reads. Took Hex board hashrate from 476 GH/s → 4.4 TH/s by
  fixing the BM1368 job-id mismatch class.
- **Per-chip stats.** Real per-ASIC hashrate / shares / errors on Hex
  boards — not just an aggregate.
- **Cyberpunk terminal dashboard (v8).** 16-component modular front-end
  on top of a pre-rendered shell. Mining-core sphere, ASIC silicon SVG,
  flow ribbon, rich block modal with solo-verification chip drill-down.
- **Tamagotchi OLED carousel.** 8-page rotating display with pixel art,
  sparklines, and live mining stats.
- **AI-native MCP server.** JSON-RPC 2.0 over HTTP at `/mcp`, 12+ tools,
  3 live resources. Drive your miner from any MCP-compatible AI client.
- **Space Heater mode.** Room-temperature targeting, BTU/h headline,
  thermostat-style control surface.
- **Advanced autotuner.** User-selectable target — max hashrate, target
  watts, target J/TH, or target temperature.
- **Home Assistant integration.** Layered: MQTT → HA auto-discovery
  (climate entity) → REST API.
- **Stratum V1 + V2.** Shared `dcentrald-stratum` crate with DCENT_OS.
- **Signed OTA.** Ed25519 verification by default, developer-mode bypass
  for tinkerers, manifest-checked update slot fit.
- **Local-first.** All dashboard assets ship with the firmware. No CDN
  fonts, no telemetry phone-home, no remote-management backdoor.

---

## Hardware support

| ASIC    | BitAxe boards                | Hashrate (typical)  | Status |
| ------- | ---------------------------- | ------------------- | ------ |
| BM1397  | Max                          | ~400 GH/s           | Verified |
| BM1366  | Ultra, Hex Ultra (6× BM1366) | ~500 GH/s / ~3 TH/s | Verified |
| BM1368  | Supra, Hex Supra (6× BM1368) | ~600 GH/s / ~3.6 TH/s | Verified |
| BM1370  | Gamma (1× BM1370), GT (2× BM1370) | ~1.2 TH/s / ~2.7 TH/s | Verified |

All six board variants run from the same Rust workspace with a build
feature per board (`--features bitaxe-gt` etc.) for the right fan
controller and ASIC count.

---

## AI-native control via MCP

DCENT_axe exposes an [MCP](https://modelcontextprotocol.io/) JSON-RPC 2.0
server at `http://<bitaxe-ip>/mcp`. The same protocol Claude and Cursor
already speak.

**Tools (callable):**

```
get_status        get_asic_info     set_frequency     set_core_voltage
set_fan_speed     set_pool          get_network       get_history
restart_mining    ota_check         get_swarm         run_autotune
```

**Resources (subscribable):**

```
bitaxe://status     live status snapshot
bitaxe://history    rolling per-minute history
bitaxe://config     current configuration
```

That means you can hand a Claude chat session your Bitaxe's IP and ask:

> *"What's the J/TH on this BitAxe right now? If it's above 22, drop the
> frequency 5%, then run an autotune for max efficiency, and let me know
> when it converges."*

…and the model will actually do it, because the protocol underneath is
a real, documented, type-checked surface — not screen-scraping.

For multi-miner fleets, the same MCP surface works against every
DCENT_axe on the LAN, and the upcoming swarm coordinator (v1.1) will
expose the fleet itself as a single MCP server.

---

## Status

DCENT_axe is at **v0.3.0 with 16 stability + UX phases shipped and
live-verified**. The recurring OOM cycle that plagues memory-tight Bitaxe
firmwares has been **eliminated** (Phase A–T), with multi-minute soaks
under sustained mining showing freeHeap drift under 15 KB.

Selected milestones from the engineering log:

- **First accepted shares** on a BitAxe Hex Supra (~4.4 TH/s) and BitAxe
  GT first-boot (~2.7 TH/s).
- **All 6 board variants verified** in v0.3.0.
- **BM1368/BM1370 job-id mismatch class fixed** — Hex Supra
  476 GH/s → 4.4 TH/s after fallback slot scan + UART frame recovery.
- **BM1370 job-id extraction mask matched to ESP-Miner** — GT
  163 GH/s → 2.7 TH/s after fixing `(id & 0xf0) >> 1` and
  `DispatcherConfig::for_bm1370()` with `job_id_step=8`.
- **TPS546 CML + phantom-overvoltage recovery** — clean recovery from
  PMBus fault states without bricking the buck regulator.
- **Coredump streaming** — chunked download of post-crash coredumps
  through the dashboard.
- **Alloc-free panic + OOM hook → NVS breadcrumb** — raw FFI
  `nvs_set_blob`, zero `format!` / `String` allocations in the panic
  path. Live-proven.
- **PSRAM enabled (+8 MB heap)** — single biggest stability win.
- **Periodic restart + heap watchdog** — defensive bounds for soak runs.
- **Streaming chips JSON + HTTP buffer pool** — eliminated >5 KB
  per-handler allocations on the hot HTTP path.
- **`/api/system/info` Serialize-derive** — 158-field DTO, JSON byte-
  identical, hammer-tested with 50 parallel pollers.
- **Modular dashboard wired end-to-end** — Phase 2.A-3.2 components,
  canonical lockup logo, Logs page, coinbase decoder feeding the
  block-tile solo-verification path.
- **Block-tile rich modal** — AGE / TXS / REWARD / DIFF / hash preview,
  LIVE pill, "Hashrate · 10m Average" caption.

---

## Build and flash

DCENT_axe is built with the **esp-rs** Rust toolchain on
ESP-IDF v5.3+ targeting `xtensa-esp32s3-espidf`.

### Build (Windows path-length workaround)

ESP-IDF requires short build paths. Use `CARGO_TARGET_DIR`:

```bash
cd projects/bitaxe-firmware
CARGO_TARGET_DIR=C:/bt cargo build --release -p dcentaxe
```

The built artifact is an **ELF** at
`C:/bt/xtensa-esp32s3-espidf/release/dcentaxe`. It is not a raw `.bin`
— flashing tools that don't understand ELF will brick the app slot.

For board-specific builds, use the matching feature:

```bash
# BitAxe GT (2× BM1370, EMC2103 fan controller)
cargo build --release -p dcentaxe \
  --no-default-features --features bitaxe-gt
```

### Flash

Three supported paths:

```bash
# espflash (preferred — handles ELF + partition table automatically)
espflash flash --port COM3 \
  --partition-table partitions.csv \
  C:/bt/xtensa-esp32s3-espidf/release/dcentaxe

# DCENT Toolbox
dcent flash --serial COM3 -f firmware.bin
dcent build-flash COM3

# OTA over Wi-Fi
# Open http://<bitaxe-ip>/ → Advanced → Firmware Update
```

`scripts/package-firmware.sh` (and the PowerShell equivalent) is the
canonical packaging gate — it reads `partitions.csv`, verifies the
update image fits the OTA app slot, and writes a manifest with
`updateFitsSlot`, `slotSize`, and SHA-256 fields. The dashboard refuses
uploads when the manifest is dishonest.

---

## Repository layout

```
bitaxe-firmware/
├── dcentaxe/             Main binary + embedded web dashboard
├── dcentaxe-asic/        BM1366 / BM1368 / BM1370 / BM1397 drivers
├── dcentaxe-hal/         Board detection, UART, I²C, power, fan, GPIO, temp
├── dcentaxe-mining/      Work dispatcher + hashrate / share tracking
├── dcentaxe-stratum/     Stratum V1 client (shared with DCENT_OS)
├── dcentaxe-stratum-v2/  Stratum V2 client
├── dcentaxe-bap/         BAP UART accessory protocol (Bitaxe Touch + DCENT_XPack)
├── dcentaxe-design-bundle/  Design system tokens shared with DCENT_OS
├── partitions.csv        Flash layout (3 MB app, 2 MB LittleFS)
├── sdkconfig.defaults    ESP-IDF config (PSRAM on, 32 KB main stack, etc.)
├── docs/                 Architecture, reviews, ship-readiness reports
├── scripts/              Package, flash, soak, and bring-up helpers
└── releases/             Release bundles + manifests
```

---

## About D-Central

[D-Central Technologies](https://d-central.tech/) is Canada's leading
Bitcoin mining technology company. Founded in 2016 in Québec, Canada.
The Bitcoin *Mining Hackers*. 2,500+ miners repaired, 400+ products,
and a stubborn belief that **every Bitcoin miner deserves to be open,
auditable, and hackable**.

DCENT_axe is part of the broader DCENT open-source ecosystem:

- [**DCENT_OS**](https://github.com/DCentralTech/DCENT_OS) — open-source
  firmware for Antminer (S9 → S21), and soon AvalonMiner and WhatsMiner.
- [**DCENT Expansion Pack**](https://github.com/DCentralTech/DCENT_ExpansionPack)
  — open-hardware Wi-Fi / OLED / temperature accessory bridge.
- **DCENT_axe** — this project.
- **DCENT Pool** — D-Central's open-source mining pool.
- **DCENT Toolbox** — Python CLI for fleet management, install, and RE.

---

## Acknowledgments

DCENT_axe is a clean-room project, but it stands on the shoulders of
public work that made the Bitaxe ecosystem what it is:

- **[BitAxe](https://bitaxe.org/)** and **[ESP-Miner](https://github.com/bitaxeorg/ESP-Miner)**
  by [@skot](https://github.com/skot) and the BitAxe community — the
  open hardware and the protocol reference that this firmware is built
  to be compatible with.
- **[Mujina](https://github.com/skygate/mujina)** — Rust-on-Zynq mining
  firmware reference work.

---

## License

DCENT_axe is licensed under [**GPL-3.0**](LICENSE). Forks, audits, and
community contributions are explicitly welcome under that license.

---

## Contributing

Bug reports and beta-tester feedback are welcome via GitHub Issues.
Pull requests should target the active development branch and include:

- The board variant you tested on (Max / Ultra / Supra / Gamma / GT /
  Hex Ultra / Hex Supra).
- A serial log from a flashed board if your change touches ASIC drivers,
  Stratum, the dispatcher, or the OTA path.
- A soak result (`soak*.json`) for changes that affect long-running
  stability — heap, dispatcher, HTTP buffer pool, or panic / OOM paths.
