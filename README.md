#short summery

--> give it a bios file (uefi)
compile it
boot it
now you have unlocked thousands of hidden bios options!!!

## Screenshots

<p align="center">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-36-02.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-36-06.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-36-17.png" width="32%">
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-36-22.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-36-30.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-01.png" width="32%">
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-02.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-03.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-08.png" width="32%">
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-12.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-24.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-28.png" width="32%">
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-36.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-37-43.png" width="32%">
  <img src="https://raw.githubusercontent.com/aryan-nikzad/SetupVarGUI/main/screenshots/Screenshot%20From%202026-09-21%2013-38-12.png" width="32%">
</p>



# long detail: SetupVar GUI + BIOS Offset Pipeline

A native UEFI application (single `.efi` binary, no OS required) for
reading/writing firmware "Setup" NVRAM variables (hidden BIOS settings),
**plus** a one-shot pipeline that scans one of your own BIOS files,
extracts the human-readable label + offset for every setting it can find,
and bakes that list into the GUI as a browsable menu — so you don't have
to hunt for offsets by hand or type hex every time.

Validated end-to-end against a real, production Gigabyte X99-PSLI BIOS
image: the pipeline correctly pulled out **516 real, correctly-labeled
settings** (e.g. `Setup,0x85,1,Non-K OC`, `Setup,0xFB,1,Extreme Memory
Profile(X.M.P.)`) straight from the firmware, and the resulting binary
boots and renders correctly in QEMU/OVMF.

> ⚠️ **This can permanently brick your motherboard if misused.** Writing
> the wrong value to the wrong offset can leave a device unbootable, with
> no software recovery possible on some boards. The pipeline only *reads*
> your BIOS file to build a lookup table — it never modifies it — but
> what you do with the resulting tool, on real hardware, is at your own
> risk. Double-check any label/offset against your firmware version
> before writing.

## Two ways to use this

### A) Just want the base tool?
`SetupVarGUI.efi` in this package works out of the box (empty
known/custom lists) — copy it to `EFI/BOOT/BOOTX64.EFI` on a FAT32 USB
stick and boot it. Use "Edit variable by name/offset" or "Browse all
NVRAM variables" manually.

### B) Want it prefilled with your BIOS's real settings?
Run the pipeline once:

```bash
./build_all.sh path/to/your_bios.rom [custom_offsets.csv] [name-filter]
```

- **`your_bios.rom`** — your motherboard's BIOS/UEFI firmware image (a
  vendor update file, or a dump/backup of your current firmware).
- **`custom_offsets.csv`** *(optional)* — your own offset/label entries,
  on top of whatever the pipeline finds automatically. See
  `custom_offsets.csv.example`.
- **`name-filter`** *(optional)* — restrict the scan to PE32 drivers whose
  name contains this text (e.g. `Setup`), for speed on firmware with
  hundreds of drivers. Default is a full scan (slower, but complete —
  many settings live in drivers not literally named "Setup").

This produces `build/SetupVarGUI.efi` with two new menus baked in:
**"Known offsets (from BIOS scan)"** (everything the pipeline found) and
**"Custom offsets"** (your CSV, if given). Selecting an entry jumps
straight to the value editor with the offset/size prefilled — you just
confirm the new value.

The pipeline does this in 4 steps, mirroring the well-known manual
workflow (see [ReBarUEFI's hidden-4G-decoding
guide](https://github.com/xCuri0/ReBarUEFI/wiki/Enabling-hidden-4G-decoding)
for the manual version of this same process):

1. **UEFIExtract** unpacks your BIOS file's firmware-volume tree.
2. **IFRExtractor-RS** runs against every PE32 driver that contains HII
   forms (not just ones literally named "Setup" — many settings live in
   platform-specific drivers), decoding the embedded IFR bytecode into
   human-readable text (`*.uefi.ifr.txt`).
3. **`pipeline/parse_ifr.py`** parses those text files for `Prompt: "..."`
   / `VarStoreInfo` / `VarOffset` / `Size` fields and the `VarStore` name
   table, producing `build/extracted_offsets.csv`.
4. **`pipeline/gen_offsets_header.py`** turns that CSV (+ your custom CSV)
   into `offsets_data.h`, a C header with two static tables, which
   `build.sh` compiles straight into the final binary.

Both `UEFIExtract` and `IFRExtractor-RS` (BSD-licensed, from
[LongSoft](https://github.com/LongSoft)) are bundled as prebuilt Linux
binaries under `tools/`; `build_all.sh` will also auto-download them if
missing.

## Files

| File / folder                    | Purpose                                              |
|-----------------------------------|-------------------------------------------------------|
| `SetupVarGUI.efi`                  | Prebuilt binary, empty known/custom lists              |
| `setupvargui.c`                    | Full GUI source                                        |
| `offsets_data.h`                   | Currently-embedded offset tables (regenerated by the pipeline; ships empty) |
| `build.sh`                         | Compiles `setupvargui.c` -> `SetupVarGUI.efi`           |
| `build_all.sh`                     | **The one-shot pipeline** described above               |
| `pipeline/parse_ifr.py`            | IFR-text -> CSV parser                                  |
| `pipeline/gen_offsets_header.py`   | CSV(s) -> `offsets_data.h` generator                     |
| `custom_offsets.csv.example`       | Template for your own offsets                           |
| `setupvar.cfg.example`             | Template for the *runtime* batch-load feature (below)   |
| `tools/uefiextract/`, `tools/ifrextractor/` | Bundled extraction tools                       |

## GUI menu overview

- **Edit variable by name/offset** — manual entry, as before.
- **Browse all NVRAM variables** — lists every NVRAM variable on the
  running machine, jump into the editor for any of them.
- **Known offsets (from BIOS scan)** — the `build_all.sh`-generated list.
- **Custom offsets** — your hand-written CSV, compiled in.
- **Load & apply config file (batch)** — a *runtime* feature (no rebuild
  needed): drop a `setupvar.cfg` next to the `.efi` file on the USB stick
  and it lets you check off entries with target *values* and apply them
  as a batch. Different from the two menus above, which just prefill the
  *offset* for you to inspect/edit — see `setupvar.cfg.example` for that
  format if you want to pre-specify values too.
- **About/disclaimer**.

## Using the built binary

1. Copy `SetupVarGUI.efi` (or `build/SetupVarGUI.efi` if you ran the
   pipeline) onto a **FAT32** USB stick as `EFI/BOOT/BOOTX64.EFI`.
2. Boot the USB stick from your firmware's boot menu (F9/F10/F11/F12 at
   power-on, or via the UEFI boot manager). Disable Secure Boot first if
   needed.
3. Navigate with arrow keys, Enter to select, Esc to go back.

## Rebuilding from source

Requires `gnu-efi` (no full EDK2 toolchain needed) and Python 3:

```bash
sudo apt-get install gnu-efi python3
./build.sh                 # plain rebuild, keeps current offsets_data.h
./build_all.sh bios.rom    # full pipeline, regenerates offsets_data.h too
```

## Offset-table format notes

`extracted_offsets.csv` / `custom_offsets.csv` columns:
`store,offset,size,label[,source]`

- `store` — NVRAM variable name (almost always `Setup`)
- `offset` — hex byte offset into that variable's data
- `size` — 1, 2 or 4 bytes (auto-extracted from the IFR bit-width where
  available; defaults to 1 otherwise — widen it in-app if needed)
- `label` — human-readable setting name, straight from the BIOS's own
  string table
- `source` — which driver it came from (informational)

`custom_offsets.csv` accepts a looser 2-column form too:
`0x495,above 4g decode` (store defaults to `Setup`, size to 1).

## Known limitations

- Offset **sizes** aren't always given explicitly in the IFR data (this
  is true of the real tool this is modeled on too); where the extractor
  can't tell, it defaults to 1 byte.
- The scan finds *labels and offsets*, not the meaning of specific
  values (what "1" vs "0" means for a given setting) — cross-check
  against the BIOS setup menu on the actual machine, or the IFR text's
  `OneOfOption` lines if you want that detail (open
  `build/extracted_offsets.csv`'s source drivers under
  `build/dump/.../*.ifr.txt` to see full option lists).
- Full-firmware scans can be slow on very large images with hundreds of
  drivers; use the `name-filter` argument to narrow it down once you know
  which driver you care about.
